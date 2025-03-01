+++
title = 'Como calcular uma primeira medição instantânea de métricas sobre indicadores cumulativos em tempo real'
date = 2025-02-28T17:18:38-03:00
draft = false
+++

Suponha que precisamos calcular dados a partir de um indicador cumulativo para gerar gráficos ou métricas em tempo real para interpretação por parte dos usuários de uma aplicação. No entanto, esse indicador é gerenciado por um processo externo, e não temos como saber exatamente quando ele é atualizado. Quais estratégias podemos adotar para processar essas informações?

Esse é um problema que encontrei em um projeto e que, a primeira vista, parece simples. No entanto, sua solução envolve algumas ideias interessantes, além de estar presente em várias funcionalidades de softwares com os quais interagimos no nosso dia a dia.

Um exemplo de aplicação que lida diretamente com esse tipo de problema, e com a qual frequentemente interagimos, são os monitores de recursos do sistema, como o Monitor do Sistema (no Linux), Gerenciador de Tarefas (no Windows), ou ferramentas de monitoramentos de recursos de servidores. Essas aplicações lidam com esse problema para calcular os gráficos que mostram o volume de tráfego de rede dentro das interfaces de rede da máquina.

![image](20250216150855.png)

Esses gráficos são geralmente construídos da seguinte forma:

O Sistema Operacional é responsável por coordenar a comunicação com o hardware. Ele processa os dados recebidos para montar os pacotes, envia-os para as aplicações, e também recebe pacotes das aplicações para serem processados e enviados ao hardware. Durante esse processo, o sistema, como intermediário de toda a comunicação, tem a capacidade de registrar estatísticas dos dados recebidos e enviados pelas interfaces de rede. Ele então acumula indicadores para refletir os dados que processou. No entanto, ele não mantém um histórico desses valores, uma vez que isso demandaria o uso crescente da memória RAM, o que eventualmente poderia sobrecarregar o sistema. Portanto ele salva apenas um indicador monotônico.

No Linux, por exemplo, o sistema contabiliza os indicadores de entrada e saída como: bytes, pacotes, erros, drops, FIFO, frames, compressão e multicast no arquivo virtual `/proc/net/dev`.

![image](20250216152007.png)

Como no caso de exemplo a intenção é somente de calcular o volume de bytes recebidos ou enviados pelo adaptador uma aplicação de monitoramento pode usar o indicador `bytes`, ele é sempre incremental, e tem duas versões que contabilizam o total de bytes recebidos e enviados pela interface desde sua inicialização.

Observando nosso indicador, vemos nossas premissas para esse problema:
- o indicador é sempre incremental
- não temos registro dos valores anteriores
- não temos controle de exatamente quando esse valor é atualizado

Com isso, é possível usar essa aplicação do problema para investigar como calcular uma métrica com as mesmas condições. Para simplificar o exemplo, vamos focar apenas na taxa de download, mas as mesmas ideias também seriam aplicáveis à taxa de upload.

Neste artigo, usando o exemplo, explorarei como gerar essas métricas e, de forma mais profunda, como abordar o cálculo do primeiro indicador em tempo real.

#### Definindo o modelo dos dados

Em primeiro lugar, o cálculo do indicador é razoavelmente simples, pois já existe um modelo de dados bem estabelecido para interpretar esse problema: as séries temporais (_time series_). Podemos definir um intervalo ∆t, que pode ser usado para capturar repetidamente os dados do indicador, e a partir desses dados construir a série temporal. Assim, cada registro da série seguiria o seguinte formato:

(timestamp, bytes atuais)

Então, a série pode ser definida como
\[
\begin{aligned}
S_{c} = (i=0,...n)\left\{ \left< t_{i},bytes_{i}\right>\right\}
\end{aligned}
\]
onde:
\[
\begin{aligned}
t_{x} = t_{0} + \Delta t \times x
\end{aligned}
\]

e i é um numero indicando o elemento atual (começando por zero)

Claro, para critérios de eficiência, não seria viável manter toda a série em memória, pois ela cresce continuamente, o que eventualmente geraria um impacto significativo no uso de memória do sistema. O uso de séries simplifica a modelagem, e com ela definida podemos só manter em memória os registros que serão necessários para os cálculos naquele momento.

A partir dessa série, podemos calcular a variação da velocidade de download em cada instante, gerando uma nova série:

\[
\begin{aligned}
S_{d} = (i=0,...n)\left\{ \left< t_{i},bytes_{i} - bytes_{i-1}\right>\right\}
\end{aligned}
\]

E a partir disso é possível personalizar a forma de atualizar o indicador de forma que ele fique mais amigável para o usuário, usando técnicas bem estabelecidas de processamento de séries temporais. Um bom exemplo para nosso caso de rede seria a de janelas móveis.

Mas nosso indicador tem um problema, que é importante em algumas situações.

#### Problemas do modelo

Veja que na nossa time series derivada dependemos da presença de \( t_{i-1} \) para calcular \( t_{i} \), mas isso dependeria da existência de um \( t_{-1} \) para calcular \( t_{0} \), o que não é possível. Para grande parte das aplicações isso não é relevante, porque poderíamos computar o indicador a partir de i = 1, No entanto, isso impede que o indicador seja exibido imediatamente para o usuário, fazendo com que ele precise esperar ∆t para ver o primeiro valor, o que pode não ser muito legal para a aplicação, como é o caso do monitor de recursos, onde o usuário espera que o programa abra instantaneamente.

Essa é a parte mais interessante desse problema sobre a qual eu não tinha encontrado nada mencionando antes, as técnicas para computar esse indicador de forma imediata. 

Apesar de nenhuma delas ser perfeita, elas exigem um olhar criativo para a solução do problema, o que leva a algumas ideias interessantes.

#### Calculando o primeiro item

A primeira técnica, que é a mais simples, mas que gera desvios significativos por não representar o estado atual, é simplesmente definir \( S_{d_{0}} = (t_{0}, 0) \), usando 0 como placeholder para primeira posição.

![image](1st-approach.gif)


Apesar da vantagem de ela ser mais simples, se o usuário que está usando a aplicação não souber desse comportamento, ele pode ter uma ideia errada de que o indicador aumentou repentinamente pouco depois de ele abrir seu monitoramento

As próximas técnicas também dependem de certa forma de manipular os dados de forma não completamente precisa, mas de uma forma que informa melhor sobre o contexto atual. 

 Mas para usar elas, primeiramente precisaríamos pensar no nosso modelo de dados de forma um pouco diferente.

#### Novo modelo de dados

Para as próximas técnicas, precisamos flexibilizar a dependência de que nossa coleta de dados sempre terá um intervalo ∆t.

Para simplificar a definição da solução, vamos primeiramente definir a função \(vS_{d}(x)\) que retorna o valor da série Sc quando i = x.

O que precisamos para as próximas abordagens é uma formula onde mesmo que o intervalo de tempo entre \(S_{c_{i}}\) e \(S_{c_{i-1}}\) usados seja diferente de Δt, ele seja comparável proporcionalmente aos demais itens da série. Uma forma de fazer isso é definindo Sd como:

\[
\begin{aligned}
S_{d} = (i=0,...n)\left\{ \left< t_{i},  (\frac{ \Delta t }{ t_{i} - t_{i-1} }) \times ( vS_{c}(i) - vS_{c}(i-1)  )   \right>\right\}
\end{aligned}
\]

Ou seja, para cada um dos itens da série \(S_{d}\), consideramos os 2 últimos itens da série \(S_{c}\), mas transformados proporcionalmente à relação entre o intervalo de tempo de coleta do indicador e o tempo padrão da série (∆t).

Com essa definição de série \(S_{d}\), podemos prosseguir para as próximas soluções.
#### Segunda solução

A segunda abordagem surge ao percebermos que existe um único instante de tempo que sabemos que existe anterior a \( t_{0} \), que é o momento em que o indicador começou a ser incrementado (e o seu valor, que é zero) dado que ele é monotônico. 

Considerando isso, podemos definir um \( S_{c_{-1}} \) imaginário com esse valor, e começar a calcular nosso \( S_{d} \) a partir de i=0. Assim, o primeiro item de nossa série consistiria na média de variação desde o início do indicador.

Usando o exemplo do cálculo da taxa de download, para essa abordagem, trocamos o `timestamp` pela variável de tempo `uptime` do sistema. O `uptime` conta quantos nanossegundos se passaram desde que o sistema foi ligado enquanto o `timestamp` conta a partir de 1 de janeiro de 1970. O `uptime` é útil para nós nessa abordagem porque ele é uma boa aproximação de quando nossa interface de rede iniciou seu funcionamento (e por consequência o indicador de bytes começou a ser incrementado). Seguindo essa estratégia poderíamos definir um \(S_{c_{-1}} = 0\) imaginário, e com isso nosso \(S_{d_{0}}\) seria efetivamente:

\[
\begin{aligned}
S_{d_{0}} = \frac{ \Delta t }{ t_{uptime} } \times vS_{c}(0)
\end{aligned}
\]

Que seria a média de uso desde que o computador foi ligado. 

Segue um exemplo de como um gráfico com sua computação pareceria em tempo real.

![image](2nd-approach.gif)

O problema desse caso é o fato de que a maior parte do contexto coletado para gerar o primeiro indicador é antigo, e pouco corresponde a realidade naquele momento. Por exemplo no gráfico acima, usando o monitoramento de rede, o usuário passou um grande período de tempo fazendo um download grande, mas agora está tendo uma carga de rede bem menor. nesse caso a velocidade de download inicial tem um desvio significativo para cima.

Ainda assim essa abordagem é frequentemente usada, por exemplo, no System Monitor (equivalente ao gerenciador de tarefas do KDE Plasma) é assim que o grafico de uso de CPU calcula sua primeira posição.

![image](20250224044328.png)
#### Terceira solução

Finalmente, a terceira abordagem (e minha favorita) para resolver o problema do primeiro ponto de dados parte da percepção humana do que consideramos como "instantâneo".

Nós humanos não conseguimos ver grande diferença entre períodos de tempo curtos, por exemplo entre algo que carrega em 100ms e algo que carrega em 300ms, mas notamos bastante essa diferença entre algo que carrega em 300ms e 900ms. Dessa forma, se tivermos um pequeno delay de até 300 milissegundos para a primeira abertura do nosso indicador, para um humano, este teria efetivamente um carregamento instantâneo em sua percepção.

Podemos usar esse tempo para calcular um indicador relativo desse período de tempo menor. Para isso capturaríamos em um período de tempo menor que Δt, e (que é na prática imperceptível para o usuário (podemos defini-lo como Δdelay) a amostra \(S_{c_{0}}\) cujo tempo seria \(t_{0} + \Delta delay\), e consideraríamos nossa base imaginária \(S_{c_{-1}}\) com a nossa amostra em \(t_{0}\)

Fazendo isso, na nossa série derivada, teriamos efetivamente na primeira posição:

\[
\begin{aligned}
S_{d_{0}} = \frac{\Delta t}{\Delta delay} \times vS_{c}( t_{0} + \Delta delay ) - vS_{c}(t_{0})
\end{aligned}
\]

Usando isso, seria possível calcular \(S_{d_{0}}\) utilizando apenas o tempo Δ*delay* ao invés de precisar esperar Δ*t*.

![image](3rd-approach.gif)

Essa abordagem tem a vantagem de que seus dados refletem uma velocidade mais atual, mas ela também tem um problema. Como não controlamos a taxa de atualização do processo interno que incrementa o indicador, nosso Δ*delay* tem que ser maior do que o pior caso de atualização desse valor, caso contrário o nosso primeiro resultado terá o comportamento da primeira abordagem.

Além disso, se o sistema de atualização dos dados tiver um intervalo entre 0,5 e 0,999... vezes o ∆delay, o resultado pode apresentar um desvio de até 50% do valor correto.

Ainda assim, essa solução é útil na maioria dos casos em que precisamos calcular o primeiro indicador em tempo real, embora não exista uma solução perfeita para esse tipo de problema. 


Por fim, como na maioria das situações de desenvolvimento, vemos mais um caso onde não existe uma solução perfeita, mas soluções com trade-offs que temos que avaliar e escolher de acordo com a natureza do problema que estamso resolvendo.


