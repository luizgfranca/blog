+++
title = 'The await from Javascript does not "wait" for a promise to resolve'
date = 2026-02-10T11:31:00-03:00
draft = false
+++

JavaScript's architecture wasn't designed from the ground up with asynchronous execution in mind, and even with the addition of features that enable asynchronicity (like `Promises` or `async/await`), this is still evident from the lack of clarity of behavior in certain situations. This leads many developers to have incorrect expectations about what will happen when their code runs. The problem is further amplified by the amount of content we have nowdays that, in well-intentioned attempts to simplify language concepts for educational purposes, end up spreading misconceptions.

This is a story about one of those behaviors where JavaScript's nature, combined with developers' understanding of how the language works, allows for less obvious problems to slip through unnoticed.

## The Context

One day I got a bug report at work on one of the systems I'd taken over. It's a batch job that converts promotion definitions from a specialized telecom product definition system to the Salesforce Commerce Cloud catalog.

These promotions can involve both attributes common to generic e-commerce promotions (like product attributes, specific product combinations, discount coupons, discount amount or percentage), as well as attributes from the currently contracted product, like associated "SVAs" (additional value-added services), product type combinations, location, and customer attributes.

The process basically works like this:
- Fetch the promotion from the product system API
- Separate attributes related to products, logged-in user data, and session-controlled data
- Search for discount coupons and select one if applicable
- Create a `Customer Group` with customer and session data requirements, and create a `Campaign` for it with the coupon rule if applicable
- Create a `Promotion` with product composition requirements
- Associate the `Promotion` with the created `Campaign`
- Save them in a result that will be used to generate XML for importing

This search to build the XML that gets imported into Salesforce Commerce Cloud is done through a class that implements the visitor design pattern, with a structure similar to this:
```js
class ProductSystemPromotionVisitor() {
	private promotions: Promotion[];
	private campaigns: Campaign[];
	private assignments: CampaignAssignment[];
	/*
	 * ...
	 */
	async visitSystemPromotion(systemPromotion: ProductSystemPromotion) {
		// ...
	}
	/*
	 * ...
	 */
	async onFinish() {
		return this.generatePromotionsXML(
			this.promotions, 
			this.campaigns, 
			this.assignments
		)
	}
}
```

The main batch process iterates through the promotions defined in the product system, and for each one found, it calls `visitSystemPromotion()` passing its definition. This method then creates the promotion and campaign objects and accumulates them in class variables. At the end of the batch process, the `onFinish()` function is called, generating the XML file with the accumulated data.

Recently, the process had been enhanced with a new feature to include this discount coupon search.

## The Problem

Given the context, we can now look at the details of the issue.

A bug was reported in which certain promotions weren't being "eligibilized" correctly.

Elibibilization is a process where the rules of promotions, properties of services selected, and characteristics of the customer are evaluated to see if the promotion should be applied or not.

When investigating this, I found that one of the promotions had a campaign that wasn't compatible with it, so I decided to investigate `visitSystemPromotion`, where this association is made.

Here's a simplified version of the method:
```ts
	async visitSystemPromotion(systemPromotion: ProductSystemPromotion) {
		if(!this.shouldGenerate(systemPromotion)) return
		
		this.promotionAttrs = this.processPomotionAttrs(systemPromotion);
		
		const couponDefinition = await this.couponRepository.findByAcronym(systemPromotion.acronym);
		this.campaignAttrs = this.processCampaignAttrs(systemPromotion);
		const coupon = this.generateCoupon(couponDefinition);
		this.campaignAttrs.coupon = coupon;
		
		const campaign = this.createCampaign(this.campaignAttrs);
		this.campaigns.push(campaign);
		
		const promotion = this.createPromotion(this.promotionAttrs);
		this.promotions.push(promotion);
		
		const assignment = this.createCampaignAssignment(campaign, promotion)
		this.assignments.push(assignment);
	}
```

Knowing that the coupon configuration feature had been done recently, it wasn't hard to figure out where to look, and consequently why the problem happened. The `await` added to load the coupons inside the `async` method that had no `await` before.

The issue with this code is caused because `await` only suspends code in the method **under** it **at least** until the `Promise` resolves. Meanwhile, other tasks can execute freely.

And that's what was happening. The campaign attributes in `this.promotionAttrs` were being replaced by another promotion's attributes since another promotion was visited while still waiting for the completion of the repository query and the next orchestration of the rest of the method.

Beyond the clear bad practice of trying to store local state inside a global object, which contributed for the bug to happen, this problem relates to a common misconception that's frequently spread when we talk about JavaScript: the claim that `await` "waits" for the result of the call it makes, which is very different from reality.

## What async/await Actually Is

`async` and `await` are really just a more ergonomic way to execute functionality that already exists in the language. In this case, this syntax is an alternative to the Promise API.

For example, considering that `a()` and `b()` are two functions that return promises

The code below:
```js
async function f() {
	await a();
	await b();
}
```

In practice executes the same functionality as this expansion using the Promise API (ignoring exception cases)
```js
function f() {
	return new Promise((resolve, _) => {
		a().then(() => {
			b().then(() => resolve())
		})
	})
}
```

This means we're only defining a simple happens-before relationship between `a()` and `b()`. Any code that's not inside this relationship is technically concurrent and can execute between the two calls.

This happens because of the mechanism the Event Loop uses to optimize the execution of code that runs concurrently. 

To avoid sitting idle while there are tasks to execute, it organizes promises that are candidates to be executed in the next tick into a queue called the "microtask queue."

When a promise can be executed, for instance in the case of a network call when the response is received from the server, the appropriate code, whether the callback defined in `then` or `catch`, is added to the end of this queue.

This architecture allows the V8 (in Node.js's case) to always stay busy with something useful unless there's no code that can be executed. This was one of the advantages that helped Node.js become popular, since it can handle multiple concurrent requests using the same OS thread to execute its logic. 

This is an approach with several advantages that was even adopted by other ecosystems, like Java and its recent addition of Virtual Threads, with the difference that Java already had better abstractions for handling concurrency, so this kind of problem is less likely to happen with it.

There's a talk that, despite being old, I watched a while back and helped me build a better mental model of how asynchronous behaviors work in Node.js, and I'd highly recommend watching it: https://www.youtube.com/watch?v=8aGhZQkoFbQ&t=31s
