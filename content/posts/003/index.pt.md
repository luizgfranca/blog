+++
title = 'Por que nem todo request fetch que respondeu realmente chamou seu servidor.'
date = 2025-06-19T16:59:50-03:00
draft = false
+++

Quando você vê esse trecho de código, quantas chamadas de rede você acredita que serão feitas quando ele for executado no navegador? (suponha que qualquer requisição feita será bem sucedida).

```js
const response1 = await fetch('http://localhost:3000/event/latest');
const data1 = await response1.json();
console.log('First request response:', data1);

const response2 = await fetch('http://localhost:3000/event/latest');
const data2 = await response2.json();
console.log('Second request response:', data2);
```

Surpreendentemente, podem ser duas, uma, ou até mesmo nenhuma!

Esse foi um caso interessante que encontrei explorando o código do Ladybird, um novo browser open source em desenvolvimento, e que achei bem interessante. O comportamento acontece porque o navegador vai decidir mandar ou não as requisições dependendo das respostas anteriores que ele recebeu do seu backend.

## Como isso acontece?

O backend (ou qualquer servidor de aplicação que possa estar executando ele, ou pelo qual suas respostas podem estar passando) pode adicionar o header `Cache-Control` com algum valor que instrui o navegador a usar o cache interno, por exemplo. `Cache-Control: max-age=10` faz com que qualquer chamada subsequente nos próximos 10 segundos use o cache ao invés de chamar o backend novamente.

Isso é um comportamento esperado da maior parte dos recursos, mas o que pode surpreender é o fato de que isso não funciona somente para páginas HTML, imagens ou arquivos JS, mas para chamadas feitas com `fetch` também.

Por exemplo, se no backend um controller for implementado para responder a `/event/latest` gerando um UUID único para cada chamada:

```js
app.get('/event/latest', (req, res) => {
    const key = uuid.v4();
    res.json({ key })
})
```

E executarmos o código acima no navegador, teremos o comportamento esperado.

![image](img20250618210010.png)
(veja que obtivemos 2 IDs diferentes).

Mas se fizermos um pequeno ajuste no backend.

```js
app.get('/event/latest', (req, res) => {
    const key = uuid.v4();
    res.header('Cache-Control', 'max-age=60')
    res.json({ key })
})
```

Teremos o seguinte resultado:
![image](img20250618210451.png)

E se recarregarmos a página antes dos pŕoximos 60 segundos:
![image](img20250618210519.png)

olhando para a aba de network veremos 4 requests:
![image](img20250618210735.png)

mas olhando para os 3 últimos...
![image](img20250618210836.png)
(note o "from disk cache").

Esse não é um comportamento exclusivo dessa composição de header. O comportamento de quaisquer regras de headers de cache (que podem ser encontrados em https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control) vai funcionar para o `fetch`, mais conceitos sobre cache também podem ser encontrados em https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching.

## Por que isso acontece?

Com a modernização das tecnologias web nos anos 2010, uma das iniciativas da W3C foi a formalização do comportamento de busca de recursos em rede, que era algo que nem sempre era tratado da mesma forma em navegadores. Um dos resultados dessa formalização foi a criação da ideia generalizada de `fetch` de `resources`, descrita na spec `fetch` (https://fetch.spec.whatwg.org/).

Uma das definições dela seria prover uma função para aplicações web que expõe esse conceito de baixo nível. E essa foi a origem da API `fetch()` que podemos chamar a partir do Javascript, e que foi introduzida em 2015 no ECMAScript 6 (ES6).

Por conta dessa origem, ela representa o mesmo conceito de busca de recursos de um serviço web que é usado por outras funcionalidades do navegador que fazem a mesma coisa, como a busca dos documentos HTML, imagens, JS, etc... E por isso compartilha o mesmo caminho no código, o que inclui o cache HTTP, que faz com que esse comportamento aconteça.

-----------
Para reproduzir os experimentos demonstrados aqui, o código usado está disponível nesse repositório do Github: https://github.com/luizgfranca/fetch-cache-example

