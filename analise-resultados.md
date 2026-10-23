# Análise dos resultados de carga

## 1. Cenário

O teste foi executado em um computador com Windows 10/11, usando Docker e
Minikube. O modelo e a capacidade do computador hospedeiro não foram
registrados nos artefatos; a capacidade documentada do cluster é de **3 CPUs e
3 GiB de memória**.

Foi usada a taxa de **1 requisição/s**, com `paradas=9`. Essa taxa foi escolhida
para medir o efeito de distribuir o cálculo entre réplicas sem saturar
deliberadamente o gerador de carga. A aplicação resolve o circuito por busca
exata (até 9! ordens) e cada Pod aceita apenas um cálculo por vez; requisições
concorrentes podem receber 503.

Cada rodada teve **10 s de aquecimento** e **60 s de medição**, além do tempo
de parada graciosa de até 10 s. Foram feitas três rodadas para cada quantidade
de réplicas (A, B e C). O cenário do k6 tinha 40 VUs pré-alocados e no máximo
40 VUs.

Cada Pod da aplicação tinha:

- request de CPU: **500m**;
- limite de CPU: **500m**;
- request de memória: **64 Mi**;
- limite de memória: **128 Mi**.

O Job do k6, separado dos Pods da aplicação, tinha request de **250m** e limite
de **500m**.

## 2. Evidências

As células abaixo são medianas das três rodadas de cada configuração. A taxa de
sucessos é `rotas_ok / 60` (tempo de medição), e não a taxa global impressa
pelo k6, que inclui o aquecimento.

| Pods | Sucessos por segundo (mediana) | Falhas (mediana) | p95 de sucesso (mediana) | Iterações descartadas (mediana) |
|---:|---:|---:|---:|---:|
| 1 | 1,00 | 0,00% | 938,96 ms | 0 |
| 2 | 1,02 | 0,00% | 939,69 ms | 0 |
| 3 | 1,02 | 0,00% | 942,42 ms | 0 |

Há uma ressalva importante na configuração de 1 Pod: a rodada A teve 6,55%
de falhas e p95 de 1.000 ms; as rodadas B e C tiveram 0% de falhas. Por isso a
mediana de falhas é 0%, mas a configuração não foi consistente em todas as
repetições.

**Distribuição e CPU observada**

- **1 Pod:** 68, 71 e 72 sucessos aceitos nas três rodadas; o Pod registrou
  entre 208m e 320m de CPU, com pico mediano de **252m**.
- **2 Pods:** distribuições de 37/35, 39/32 e 33/38 sucessos por Pod,
  respectivamente. O balanceamento ficou próximo de metade para cada Pod,
  sem recusas. Os picos agregados observados foram 121m, 392m e 141m por
  rodada (mediana de **141m**); individualmente, os picos variaram de 17m a
  215m.
- **3 Pods:** distribuições de 23/19/28, 25/22/25 e 18/31/23 sucessos por
  Pod. Também não houve recusas, e a distribuição permaneceu razoavelmente
  espalhada, embora menos uniforme na rodada C. Os picos agregados observados
  foram 332m, 243m e 401m (mediana de **332m**); individualmente, os picos
  variaram de 4m a 160m.

Em todas as configurações, `dropped_iterations{scenario:medicao}` foi zero.
As metas do projeto eram p95 < 800 ms, falhas < 1% e nenhuma iteração
descartada.

## 3. Explicação

### Observado

- Passar de 1 para 2 Pods eliminou a falha da rodada A e manteve as falhas em
  0% nas três rodadas de 2 Pods.
- 2 e 3 Pods tiveram praticamente a mesma vazão: medianas de 1,02
  sucessos/s.
- O p95 não caiu para menos de 800 ms: ficou em aproximadamente 939--942 ms
  com 2 ou 3 Pods.
- A distribuição entre Pods confirma que o Service encaminhou trabalho para
  mais de uma réplica, sem um Pod concentrar todos os sucessos.
- A adição do terceiro Pod não aumentou a vazão nem reduziu a latência; o p95
  mediano foi ligeiramente maior que com 2 Pods.

### Hipótese

O primeiro Pod ajuda principalmente porque cada processo só pode executar um
cálculo por vez. Com 2 Pods, uma requisição pode ser processada enquanto outra
usa a segunda vaga, reduzindo recusas quando há sobreposição. O ganho deixa de
aparecer com 3 Pods porque o gargalo provável passa a ser o próprio cálculo
CPU-bound de uma rota de 9 paradas, o limite de 500m por Pod ou a sobrecarga
do caminho Service/HTTP, e não apenas a quantidade de vagas.

Também é possível que o custo fixo de cada requisição e a variabilidade do
escalonamento no cluster pequeno mascarem qualquer ganho do terceiro Pod. Isso
é uma hipótese, não uma conclusão isolada pelos dados atuais: as medições de
CPU são amostras de `kubectl top`, não um perfil do código nem uma série
contínua de utilização.

## 4. Decisão

Para essa demanda, eu adotaria **2 Pods como configuração provisória**:
eles eliminaram as falhas observadas, distribuíram o trabalho de forma
razoavelmente equilibrada e não custaram o terceiro Pod, que não trouxe ganho
mensurável de vazão ou p95.

O custo reservado para a aplicação é:

- 2 Pods × 500m = **1.000m, ou 1 CPU reservada**;
- além disso, o Job do gerador reserva 250m enquanto está executando.

Essa decisão não significa que a meta foi atingida. Nenhuma das três
configurações atingiu simultaneamente todas as metas, principalmente por causa
do p95 acima de 800 ms. A próxima investigação deve medir o gargalo por etapa:
profiling do `calcular_rota`, verificação de throttling no limite de 500m e um
teste controlado aumentando a CPU reservada/limite por Pod (por exemplo, 1
CPU), mantendo a mesma taxa e duração. Se o tempo do cálculo dominar, a
correção mais promissora é otimizar ou substituir a busca exaustiva antes de
continuar aumentando réplicas.
