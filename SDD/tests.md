# Testes — Zona Azul Digital

## 1. Objetivo e execução por TDD

Especificar os testes automatizados que deverão ser implementados para verificar spec.md e o contrato HTTP de plan.md.

Para cada comportamento, escrever e executar o teste antes da implementação, confirmar a falha pelo comportamento ausente (Red), implementar o necessário (Green) e refatorar mantendo todos os testes passando (Refactor).

Este documento descreve cenários; não contém implementação de testes.

## 2. Preparação e isolamento

- Criar uma aplicação com repositório em memória vazio, contador iniciado em 1 e lock próprio para cada teste independente.
- Compartilhar esse repositório entre as requisições de um mesmo cenário.
- Substituir o relógio por uma dependência controlável. Não usar esperas reais para simular estacionamento.
- Fixar, salvo indicação contrária, a entrada em 2026-10-07T08:00:00-03:00 e avançar o relógio até a saída desejada.
- Usar ABC1D23 como placa principal e XYZ9A87 como segunda placa válida.
- Usar os identificadores retornados na abertura nas operações seguintes.
- Executar com tarifa de 600 centavos por hora, fração de 30 minutos, teto de 5000 centavos, tolerância de 10 minutos e porta 8005.
- Nos testes HTTP, conferir código HTTP, conteúdo, tipos e ausência de campos indevidos. Cada erro previsto deve conter somente o campo erro.
- Nos casos de cobrança, executar tanto testes unitários do cálculo quanto testes HTTP do encerramento.

## 3. Abertura e validação — UC1

| ID | Preparação e ação | Resultado esperado |
| --- | --- | --- |
| AB01 | Abrir ABC1D23 sem entrada, com relógio fixo às 08:00 | HTTP 201; id inteiro positivo; placa ABC1D23; status aberto; entrada no instante fixado, com fuso -03:00 |
| AB02 | Informar entrada 2026-10-07T07:00:00-03:00 com relógio às 08:00 | HTTP 201; preservar o instante informado, sem substituí-lo pelo relógio |
| AB03 | Informar entrada 2026-10-07T11:00:00Z | HTTP 201; retornar o instante equivalente a 2026-10-07T08:00:00-03:00 |
| AB04 | Omitir placa | HTTP 422; erro placa_invalida; nenhum bilhete criado |
| AB05 | Enviar placa vazia, com 6 caracteres ou com 8 caracteres, em testes separados | HTTP 422; erro placa_invalida |
| AB06 | Enviar abc1d23, ABC-123, ABC1D2 espaço ou ÁBC1D23, em testes separados | HTTP 422; erro placa_invalida; não normalizar a placa |
| AB07 | Enviar placa como null, número, booleano, lista ou objeto | HTTP 422; erro placa_invalida; não converter tipos |
| AB08 | Enviar entrada como texto inválido ou data impossível | HTTP 422; erro entrada_invalida |
| AB09 | Enviar entrada 2026-10-07T08:00:00 sem fuso | HTTP 422; erro entrada_invalida |
| AB10 | Enviar entrada como null, número ou booleano | HTTP 422; erro entrada_invalida |
| AB11 | Enviar placa inválida e entrada inválida juntas | HTTP 422; erro placa_invalida, conforme a precedência adotada no plano |
| AB12 | Com ABC1D23 já aberta, enviar nova abertura com entrada inválida | HTTP 422; erro entrada_invalida, antes do conflito de placa ocupada |
| AB13 | Abrir duas placas diferentes válidas | Duas respostas 201 e identificadores distintos |

Para AB01 a AB03, conferir exatamente os campos id, placa, entrada e status. Representações ISO-8601 equivalentes com fuso -03:00 são aceitas.

## 4. Cobrança, tolerância e teto — UC2 e UC7

Preparação: abrir um bilhete com a entrada fixa e encerrar com relógio controlado na duração indicada. Cada linha é um cenário independente. Esperar HTTP 200.

| ID | Duração exata | minutos esperado | valor_centavos esperado | Regra verificada |
| --- | --- | ---: | ---: | --- |
| CB01 | 0 minutos | 0 | 0 | Duração zero |
| CB02 | 9 minutos | 9 | 0 | Abaixo da tolerância |
| CB03 | 10 minutos | 10 | 0 | Limite gratuito inclusivo |
| CB04 | 11 minutos | 11 | 300 | Um minuto acima da tolerância |
| CB05 | 29 minutos | 29 | 300 | Abaixo da primeira fração |
| CB06 | 30 minutos | 30 | 300 | Fração exata |
| CB07 | 31 minutos | 31 | 600 | Fração seguinte e ausência de desconto da tolerância |
| CB08 | 60 minutos | 60 | 600 | Hora cheia |
| CB09 | 61 minutos | 61 | 900 | Adjacência à hora cheia |
| CB10 | 95 minutos | 95 | 1200 | Quatro frações na variante |
| CB11 | 480 minutos | 480 | 4800 | Última fração abaixo do teto |
| CB12 | 481 minutos | 481 | 5000 | Cobrança calculada de 5100 limitada ao teto |
| CB13 | 510 minutos | 510 | 5000 | Fração exata acima do teto |
| CB14 | 1440 minutos | 1440 | 5000 | Permanência de 24 horas |
| CB15 | 2880 minutos | 2880 | 5000 | Teto não multiplicado por dois dias |
| CB16 | 10 minutos e 1 segundo | 11 | 300 | Segundos excedentes, conforme decisão do spec.md |
| CB17 | 30 minutos e 1 segundo | 31 | 600 | Nova fração por segundos excedentes |

> [!WARNING]
> Nenhum cenário pode retornar cobrança superior a 5000 centavos. O valor de 1250 centavos do exemplo genérico não corresponde aos 95 minutos desta variante: o resultado correto é 1200.

Em todos os encerramentos bem-sucedidos, exigir exatamente id, placa, entrada, saida, minutos e valor_centavos. Conferir que minutos e valor_centavos sejam inteiros JSON, não strings, booleanos ou números decimais. Não aceitar o campo valor.

## 5. Encerramento e integridade de estado — UC2

| ID | Preparação e ação | Resultado esperado |
| --- | --- | --- |
| EN01 | Encerrar bilhete aberto sem corpo na requisição | HTTP 200; saída corresponde ao relógio; bilhete passa a encerrado |
| EN02 | Encerrar identificador inexistente | HTTP 404; erro bilhete_nao_encontrado |
| EN03 | Encerrar novamente bilhete já encerrado após avançar o relógio | HTTP 409; erro bilhete_ja_encerrado; saída, duração e valor originais preservados |
| EN04 | Cancelar e tentar encerrar o mesmo bilhete | HTTP 409; erro bilhete_nao_aberto; preservar cancelamento, sem saída ou cobrança |
| EN05 | Consultar ativos e histórico após encerrar | Ausente dos ativos; presente no histórico como encerrado, com dados de encerramento |

EN04 verifica uma decisão explicitada no spec.md e no plan.md para uma lacuna do contrato.

## 6. Cancelamento — UC5

| ID | Preparação e ação | Resultado esperado |
| --- | --- | --- |
| CA01 | Cancelar bilhete aberto sem corpo | HTTP 200; id, placa e entrada preservados; status cancelado |
| CA02 | Inspecionar resposta e histórico do cancelado | Ausência de saida, minutos e valor_centavos; não aceitar esses campos como null ou zero |
| CA03 | Cancelar identificador inexistente | HTTP 404; erro bilhete_nao_encontrado |
| CA04 | Cancelar bilhete encerrado | HTTP 409; erro bilhete_nao_aberto; dados anteriores preservados |
| CA05 | Cancelar bilhete já cancelado | HTTP 409; erro bilhete_nao_aberto; estado preservado |
| CA06 | Cancelar após 481 minutos de permanência | HTTP 200; sem cobrança, apesar da duração |

## 7. Uma vaga por placa e concorrência — UC8

| ID | Preparação e ação | Resultado esperado |
| --- | --- | --- |
| VP01 | Abrir novamente uma placa com bilhete aberto | HTTP 409; erro bilhete_em_aberto; apenas um registro no histórico |
| VP02 | Encerrar e abrir novamente a mesma placa | HTTP 201; novo id; anterior permanece encerrado no histórico |
| VP03 | Cancelar e abrir novamente a mesma placa | HTTP 201; novo id; anterior permanece cancelado no histórico |
| VP04 | Enviar duas aberturas simultâneas válidas para a mesma placa livre | Uma resposta 201 e outra 409 com bilhete_em_aberto; exatamente um bilhete aberto e um registro criado |
| VP05 | Encerrar e cancelar simultaneamente um bilhete aberto | Somente uma operação retorna 200; a outra retorna 409 com bilhete_nao_aberto; estado final consistente com a operação aceita |
| VP06 | Enviar dois encerramentos simultâneos | Uma resposta 200 e outra 409 com bilhete_ja_encerrado; um único encerramento registrado |

Usar sincronização de início das operações nos testes concorrentes, sem depender de qual solicitação será atendida primeiro. Conferir o resultado e o estado final, não apenas a ausência de exceções.

## 8. Consulta de ativos — UC3

| ID | Preparação e ação | Resultado esperado |
| --- | --- | --- |
| AT01 | Consultar repositório vazio | HTTP 200; array vazio |
| AT02 | Preparar um aberto, um encerrado e um cancelado e consultar | Somente o aberto aparece |
| AT03 | Abrir placas distintas com entradas às 08:00 e 09:00 | Entrada das 09:00 aparece primeiro |
| AT04 | Criar primeiro entrada às 09:00 e depois entrada às 08:00 | Ordenação continua pela entrada, não pela ordem de criação |
| AT05 | Abrir placas distintas com entrada idêntica | Maior id aparece primeiro, conforme decisão do spec.md |
| AT06 | Encerrar ou cancelar o único aberto e consultar | Array vazio |

Cada item deve conter id, placa, entrada e status aberto.

## 9. Histórico por placa — UC6

| ID | Preparação e ação | Resultado esperado |
| --- | --- | --- |
| HI01 | Consultar placa válida sem bilhetes | HTTP 200; array vazio |
| HI02 | Para a mesma placa, criar um encerrado, depois um cancelado e depois um aberto | Retornar os três registros com os respectivos estados |
| HI03 | Preparar bilhetes de duas placas e consultar apenas uma | Retornar somente os da placa consultada |
| HI04 | Preparar entradas distintas no histórico | Ordenar por entrada decrescente |
| HI05 | Preparar entradas iguais no histórico | Desempatar por id decrescente |
| HI06 | Omitir placa ou informar formato inválido | HTTP 422; erro placa_invalida |
| HI07 | Inspecionar registros dos três estados | Todos têm id, placa, entrada e status; somente encerrados têm saida, minutos e valor_centavos |

## 10. Relatório diário — UC4

Cada cenário parte de repositório isolado. Os horários e datas consideram o fuso -03:00. Os valores devem ser obtidos pelas operações normais de abertura e encerramento, controlando o relógio.

| ID | Preparação e consulta | Resultado esperado |
| --- | --- | --- |
| RE01 | Consultar 2026-10-07 sem encerramentos | HTTP 200; data 2026-10-07; total_bilhetes 0; faturamento_centavos 0; tempo_medio_minutos 0 |
| RE02 | Encerrar no dia dois bilhetes de 30 e 31 minutos | total_bilhetes 2; faturamento_centavos 900; tempo_medio_minutos 31 |
| RE03 | Encerrar no dia bilhetes de 30, 30 e 31 minutos | total_bilhetes 3; faturamento_centavos 1200; tempo_medio_minutos 30 |
| RE04 | Encerrar no dia bilhetes de 30, 31 e 31 minutos | total_bilhetes 3; faturamento_centavos 1500; tempo_medio_minutos 31 |
| RE05 | Encerrar somente um bilhete de 10 minutos | total_bilhetes 1; faturamento_centavos 0; tempo_medio_minutos 10 |
| RE06 | Encerrar bilhetes de 10 e 11 minutos no dia | total_bilhetes 2; faturamento_centavos 300; tempo_medio_minutos 11 |
| RE07 | Ter um encerrado de 30 minutos, um aberto e um cancelado | total_bilhetes 1; faturamento_centavos 300; tempo_medio_minutos 30 |
| RE08 | Abrir em 2026-10-06 às 23:50 e encerrar em 2026-10-07 às 00:20 | No dia 07: quantidade 1, faturamento 300 e média 30; no dia 06: métricas zero |
| RE09 | Abrir em 2026-10-06 às 23:30 e encerrar exatamente em 2026-10-07 às 00:00 | Incluir no dia 07 e excluir do dia 06 |
| RE10 | Abrir em 2026-10-07 às 23:30 e encerrar em 2026-10-08 às 00:00 | Excluir do dia 07 e incluir no dia 08 |
| RE11 | Abrir em 2026-10-07 às 22:00 e encerrar às 22:30, equivalentes a 01:30Z do dia 08 | Incluir no relatório do dia 07, conforme data local |
| RE12 | Encerrar dois bilhetes de 481 minutos no mesmo dia | total_bilhetes 2; faturamento_centavos 10000; tempo_medio_minutos 481; teto não limita o faturamento agregado |
| RE13 | Omitir data | HTTP 422; erro data_invalida |
| RE14 | Informar 07/10/2026, 2026-2-03 ou texto arbitrário | HTTP 422; erro data_invalida |
| RE15 | Informar 2026-02-30 ou 2026-13-01 | HTTP 422; erro data_invalida |

Conferir exatamente os campos data, total_bilhetes, faturamento_centavos e tempo_medio_minutos. As três métricas devem ser inteiros.

A contagem de encerrados e o retorno de zero para dia vazio seguem as decisões de negócio do spec.md.

## 11. Armazenamento e isolamento

| ID | Ação | Resultado esperado |
| --- | --- | --- |
| ME01 | Abrir um bilhete e consultá-lo em requisição posterior da mesma aplicação | Registro preservado |
| ME02 | Criar uma nova instância independente da aplicação e do repositório | Nenhum bilhete da instância anterior presente |
| ME03 | Criar bilhetes, encerrar ou cancelar e criar outros | Identificadores distintos, sem reutilização |
| ME04 | Alterar a cópia de um registro retornada pelo repositório em teste unitário | Registro armazenado permanece inalterado |
| ME05 | Executar os testes em ordem diferente | Mesmos resultados; ausência de dependência entre cenários |

## 12. Inicialização e container

| ID | Ação | Resultado esperado |
| --- | --- | --- |
| EX01 | Iniciar a aplicação sem fornecer variáveis de ambiente | Serviço disponível na porta 8005 com os parâmetros da variante |
| EX02 | Construir o Containerfile e executar publicando 8005 para 8005 | API acessível em http://localhost:8005 |
| EX03 | Abrir e consultar bilhete pela API do container | Abertura 201 e registro disponível na consulta |
| EX04 | Reiniciar o processo e consultar ativos | Array vazio, conforme armazenamento volátil definido no plano |
| EX05 | Verificar configuração de inicialização | Um worker, sem recarga automática, servidor em 0.0.0.0:8005 |

## 13. Critérios de aprovação

- Todos os cenários aplicáveis devem ser implementados e passar.
- Os testes devem comprovar resultados esperados e efeitos sobre o estado.
- Não utilizar esperas reais para reproduzir durações de estacionamento.
- A suíte deve rodar sem banco de dados ou serviço externo.
- Conferir cobertura dos oito casos de uso, das regras de arredondamento, dos conflitos e do contrato HTTP.
- Registrar na documentação gerada como executar a suíte e o linter.
