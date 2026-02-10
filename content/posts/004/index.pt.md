+++
title = 'O await do javascript não "espera" que sua Promise se complete'
date = 2026-02-10T11:31:00-03:00
draft = false
+++

A arquitetura do Javascript não foi projetada desde o começo com execução assíncrona em mente, e mesmo com a adição de recursos que permitem assincronia (como `Promises` ou `async/await`) ainda é claro isso pela falta de clareza sobre o comportamento em alguns momentos, o que faz com que vários desenvolvedorres da linguagem tenham expectativas incorretas sobre o que vai acontecer com seu código quando ele for executado. O problema ainda é amplificado pela quantidade de conteúdo que temos hoje em dia que, na tentativa bem intencionada de simplificar conceitos da linguagem por motivos didáticos, acabam propagando ideias incorretas.

Essa é uma história de um desses comportamentos onde a natureza do Javascript, e o entendimento dos desenvolvedores que usam a linguagem sobre como ela funciona permitem que problemas menos óbvios passem despercebidos.

## O contexto

Um dia me relataram um bug no trabalho em um dos sistemas que eu tinha assumido. Ele é um batch job que converte definições de promoção de um sistema de definição de produtos especializado de telecomunicações para o catalogo do Salesforce Commerce Cloud. 

Essas promoções podem envolver tanto atributos que são comuns de promoções em e-commerce genéricos, como atributos dos produtos, combinações de produtos específicos, como cupons de desconto, valor ou percentual de desconto, quanto atributos do produto atualmente contratado, como SVAs associados, combinações de tipos de produtos, localidade e atributos de clientes.

Esse processo funciona basicamente da seguinte forma:
- Obtenção da promoção na API do sistema de produtos
- Separação dos atributos relacionados aos produtos, dados do usuário logado e dados controlados por sessão
- buscar cupons de desconto e selecionar algum se aplicável
- criar `Customer Group` com requisitos de dados de cliente e sessão, e criar uma `Campaign` para ele com a regra de cupom se aplicável
- criar `Promotion` com requisitos de composição de produtos
- associar `Promotion` com `Campaign` criados
- salva-los em resultado que será usado para gerar o XML de importação

Essa busca para montagem do XML que é importado no Salesforce Commerce Cloud é feita a partir de uma classe que implementa o design pattern de visitor, e tem uma estrutura semelhante a essa:

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

Onde o processo principal de batch navega as promoções definidas no sistema de produtos, e para cada uma encontrada chama o `visitSystemPromotion()` passando sua definição. Esse método então cria os objetos de promoção e campanha, e acumula em variáveis da classe. Ao final do processo de batch a função `onFinish()` é chamada, gerando o arquivo XML com os dados acumulados.

Recentemente o processo tinha sido evoluído com uma nova feature para incluir essa busca de cupom de desconto.
## O problema

Dado o contexto podemos agora olhar os detalhes do problema:

Foi relatado um bug onde certas promoções não estavam sendo eligibilizadas corretamente, ao investigar o problema vi que uma das promoções estava com uma campanha que não era compatível com ela, então resolvi investigar a `visitSystemPromotion`, onde essa associação é feita.

Segue uma versão simplificada do método:
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

Sabendo que a feature de configuração de cupons tinha sido feita recentemente não foi tão dificil saber para onde olhar, e consequentemente porque o problema aconteceu.

O problema desse código é causado porque o `await` somente suspende codigo do método que está **abaixo** dele **pelo menos** até que a `Promise` seja resolvida, enquanto isso outras tarefas podem ser executadas, e era o que acontecia, os atributos da campanha de `this.promotionAttrs` foram substituídos pelo de outra uma vez que outra promoção foi visitada enquanto ainda se aguardava a proxima orquestração do restante do método.

Além da má prática clara que existia ali de tentar guardar estados locais dentro de um objeto global, o que inflenciou o bug esse problema tem a ver com uma concepção errada que é frequentemente propagada quando falamos de javscript, que é a alegação de que o `await` "espera" o resultado da chamada feita por ele, sendo que isso é muito diferente da realidade. 

## O que realmente é o async/await

`async` e `await` são, na realidade, apenas uma forma mais ergonômica de executar uma funcionalidade já existente na linguagem. Nesse caso essa sintaxe é uma alternativa à API de promises.

Por exemplo, considerando que `a()` e `b()` são duas funções que retornam promises, o código abaixo
```js
async function f() {
	await a();
	await b();
}
```

Na prática executa a mesma funcionalidade que essa expansão usando a API de promises (ignorando o caso de exceções)
```js
function f() {
	return new Promise((resolve, _) => {
		a().then(() => {
			b().then(() => resolve())
		})
	})
}
```

Isso quer dizer que estamos definindo só uma simples relação de happens-before de `a()` para `b()`, qualquer código que não está dentro dessa relação tecnicamente é concorrente, e pode executar entre as duas chamadas.

Isso acontece por conta do mecanismo que o Event Loop usa para otimizar a execução de códigos que executam de forma concorrente. Para evitar ficar ocioso enquanto existem tarefas para executar ele organiza as promises que são candidatas a serem executadas no proximo tick dentro de uma fila chamada de "microtask queue". 

No momento em que uma promise pode ser executada, por exemplo no caso de uma chamada de rede quando a resposta foi recebida do servidor, o código adequado, seja o callback definido no `then` ou seja o definido no `catch` é adicionado ao final dessa fila. 

Esta arquitetura permite que a não ser que não exista nenhum código que pode ser executado, a V8 (no caso do Node.js) pode sempre continuar ocupada com algo útil, e foi uma das vantagens que ajudou o Node.js a se tornar popular, pois ele pode lidar com várias requisições concorrentes usando a mesma thread do sistema operacional para executar sua lógica, abordagem com várias vantage, que até foi adotada por outros ecossistemas, como Java e sua recente adição de Virtual Threads, com uma diferença de que o Java já possuia abstrações melhores para lidar com concorrência.

Existe uma palestra que apesar de antiga, eu vi um tempo atrás e me ajudou a criar um modelo mental melhor de como comportamentos assíncronos funcionam no Node.js, e eu recomendaria muito assistir: https://www.youtube.com/watch?v=8aGhZQkoFbQ&t=31s
