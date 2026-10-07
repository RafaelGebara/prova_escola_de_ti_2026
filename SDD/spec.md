# Especificação de negócio — Zona Azul Digital

## 1. Objetivo

Gerenciar bilhetes de estacionamento rotativo, permitindo abertura,
encerramento, cancelamento, consulta de bilhetes ativos, histórico
por placa e relatório diário.

## 2. Regras de cobrança

| Regra | Valor |
| --- | --- |
| Tarifa por hora | R$ 6,00 |
| Fração de cobrança | 30 minutos |
| Valor por fração | R$ 3,00 |
| Tolerância gratuita | 10 minutos |
| Limite de cobrança por bilhete | R$ 50,00 |

Durações de até 10 minutos, inclusive, são gratuitas.

Ultrapassada a tolerância, cobrar todo o período desde a entrada,
sem descontar os 10 minutos iniciais.

Cada fração de 30 minutos iniciada deve ser cobrada integralmente.

> [!WARNING]
> O valor máximo é R$ 50,00 por bilhete, mesmo quando a permanência
> atravessa dias. A mudança de dia não reinicia a cobrança.

## 3. UC1 — Abrir bilhete

### Regras de negócio

- A abertura deve identificar a placa do veículo.
- A placa deve conter exatamente 7 caracteres, formados por letras
  maiúsculas de A a Z ou números de 0 a 9.
- Placas ausentes ou inválidas impedem a abertura.
- Cada bilhete deve ter uma identificação única.
- Um novo bilhete deve iniciar no estado aberto.
- Quando uma entrada for informada, ela deve ser utilizada.
- Quando não houver entrada informada, considerar o momento da abertura.
- Uma entrada inválida impede a abertura.
- Uma placa com bilhete aberto não pode abrir outro.

### Critérios de aceite

| ID | Situação | Resultado esperado |
| --- | --- | --- |
| AC1.1 | Abrir bilhete para ABC1D23 sem outro aberto | Criar bilhete identificado e em estado aberto |
| AC1.2 | Abrir sem informar o momento de entrada | Registrar o momento da abertura |
| AC1.3 | Informar um momento de entrada válido | Registrar o momento informado |
| AC1.4 | Informar placa ausente, com minúsculas, símbolos ou tamanho diferente de 7 | Rejeitar abertura |
| AC1.5 | Informar entrada inválida | Rejeitar abertura sem criar bilhete |

## 4. UC2 — Encerrar bilhete

### Regras de negócio

- Somente bilhetes abertos podem ser encerrados.
- O encerramento deve registrar o momento de saída.
- A duração deve corresponder ao período entre entrada e saída.
- O valor deve respeitar a tolerância, as frações e o teto por bilhete.
- Após o encerramento, o bilhete passa ao estado encerrado.
- Um bilhete encerrado não deve aparecer entre os ativos.
- Uma nova tentativa de encerramento não deve alterar a saída
  nem o valor já registrado.
- Não é possível encerrar um bilhete inexistente.

### Critérios de aceite

As durações abaixo correspondem a minutos exatos.

| ID | Situação | Resultado esperado |
| --- | --- | --- |
| AC2.1 | Encerrar após 30 minutos | Cobrar R$ 3,00 |
| AC2.2 | Encerrar após 31 minutos | Cobrar R$ 6,00 |
| AC2.3 | Encerrar após 60 minutos | Cobrar R$ 6,00 |
| AC2.4 | Encerrar após 61 minutos | Cobrar R$ 9,00 |
| AC2.5 | Encerrar após 95 minutos | Cobrar R$ 12,00 |
| AC2.6 | Encerrar após 480 minutos | Cobrar R$ 48,00 |
| AC2.7 | Encerrar após 481 minutos | Cobrar R$ 50,00 |
| AC2.8 | Encerrar após 48 horas | Cobrar R$ 50,00 |
| AC2.9 | Encerrar bilhete aberto | Registrar saída, duração e cobrança; mudar para encerrado |
| AC2.10 | Encerrar novamente bilhete encerrado | Rejeitar e preservar os dados anteriores |
| AC2.11 | Encerrar bilhete inexistente | Rejeitar a operação |

## 5. UC3 — Consultar bilhetes ativos

### Regras de negócio

- A consulta deve apresentar somente bilhetes abertos.
- Bilhetes encerrados e cancelados não são ativos.
- Os bilhetes devem aparecer da entrada mais recente para a mais antiga.
- Quando não houver bilhetes ativos, a consulta deve apresentar
  uma lista vazia.

### Critérios de aceite

| ID | Situação | Resultado esperado |
| --- | --- | --- |
| AC3.1 | Não existem bilhetes abertos | Lista vazia |
| AC3.2 | Existem bilhetes abertos, encerrados e cancelados | Exibir somente os abertos |
| AC3.3 | Existem entradas às 08:00 e às 09:00 no mesmo dia | Exibir primeiro a entrada das 09:00 |
| AC3.4 | Um bilhete acaba de ser encerrado ou cancelado | Não exibi-lo entre os ativos |

## 6. UC4 — Emitir relatório diário

### Regras de negócio

- A consulta deve indicar uma data válida.
- O relatório deve apresentar a data consultada, a quantidade
  de bilhetes, o faturamento e o tempo médio de permanência.
- O faturamento corresponde aos valores dos bilhetes encerrados no dia.
- O tempo médio considera somente bilhetes encerrados no dia.
- Bilhetes encerrados gratuitamente também participam da média.
- A média deve ser arredondada para o minuto inteiro mais próximo,
  com 0,5 arredondado para cima.
- O dia de referência deve seguir o fuso -03:00.

O recorte da quantidade de bilhetes e o resultado para dias sem
encerramentos são decisões de negócio explicitadas na seção 11.

### Critérios de aceite

| ID | Situação | Resultado esperado |
| --- | --- | --- |
| AC4.1 | Dois bilhetes encerrados no dia, com durações de 30 e 31 minutos | Quantidade 2, faturamento R$ 9,00 e média de 31 minutos |
| AC4.2 | Dia sem bilhetes encerrados | Quantidade, faturamento e média iguais a zero |
| AC4.3 | Existem bilhetes abertos ou cancelados | Não entram no faturamento nem na média |
| AC4.4 | Bilhete aberto no dia anterior e encerrado na data consultada | Participa do relatório da data de encerramento |
| AC4.5 | Bilhete encerrado à meia-noite do dia seguinte | Não participa do relatório do dia anterior |
| AC4.6 | Data ausente ou inexistente, como 30 de fevereiro | Rejeitar consulta |
| AC4.7 | Único encerramento do dia com duração de 10 minutos | Quantidade 1, faturamento zero e média de 10 minutos |

## 7. UC5 — Cancelar bilhete

### Regras de negócio

- Somente bilhetes abertos podem ser cancelados.
- O cancelamento altera o estado para cancelado.
- Cancelar não gera cobrança nem registra saída.
- O bilhete cancelado permanece no histórico.
- Após cancelar, a placa pode abrir outro bilhete.
- Não é possível cancelar bilhete inexistente, encerrado
  ou já cancelado.

### Critérios de aceite

| ID | Situação | Resultado esperado |
| --- | --- | --- |
| AC5.1 | Cancelar bilhete aberto | Estado cancelado, sem saída e sem cobrança |
| AC5.2 | Cancelar bilhete inexistente | Rejeitar operação |
| AC5.3 | Cancelar bilhete encerrado | Rejeitar e preservar o encerramento |
| AC5.4 | Cancelar novamente bilhete cancelado | Rejeitar e preservar o cancelamento |
| AC5.5 | Abrir novo bilhete após cancelamento | Permitir abertura e manter o anterior no histórico |

## 8. UC6 — Consultar histórico por placa

### Regras de negócio

- A consulta exige uma placa válida.
- O histórico deve apresentar todos os bilhetes da placa,
  independentemente do estado.
- Não deve apresentar bilhetes de outras placas.
- A ordenação deve ser da entrada mais recente para a mais antiga.
- Uma placa sem bilhetes deve apresentar histórico vazio.

### Critérios de aceite

| ID | Situação | Resultado esperado |
| --- | --- | --- |
| AC6.1 | Consultar placa que nunca estacionou | Histórico vazio |
| AC6.2 | Placa possui bilhetes abertos, encerrados e cancelados | Exibir todos |
| AC6.3 | Existem bilhetes de outras placas | Não incluí-los |
| AC6.4 | Há bilhetes com diferentes momentos de entrada | Exibir os mais recentes primeiro |
| AC6.5 | Placa ausente ou inválida | Rejeitar consulta |

## 9. UC7 — Aplicar tolerância gratuita

### Regras de negócio

- Cada bilhete possui tolerância própria de 10 minutos.
- Permanências de até 10 minutos são gratuitas.
- Ultrapassar a tolerância torna toda a permanência cobrável.
- A tolerância não deve ser subtraída da duração.

### Critérios de aceite

| ID | Duração exata | Resultado esperado |
| --- | --- | --- |
| AC7.1 | 0 minutos | Gratuito |
| AC7.2 | 9 minutos | Gratuito |
| AC7.3 | 10 minutos | Gratuito |
| AC7.4 | 11 minutos | R$ 3,00 |
| AC7.5 | 31 minutos | R$ 6,00, sem desconto dos 10 minutos |

## 10. UC8 — Garantir uma vaga por placa

### Regras de negócio

- Uma placa pode possuir no máximo um bilhete aberto.
- Tentativas simultâneas de abertura não podem gerar dois
  bilhetes abertos para a mesma placa.
- Bilhetes encerrados ou cancelados não impedem nova abertura.
- Uma abertura rejeitada não deve acrescentar bilhetes ao histórico.

### Critérios de aceite

| ID | Situação | Resultado esperado |
| --- | --- | --- |
| AC8.1 | Abrir bilhete para placa que já possui um aberto | Rejeitar nova abertura |
| AC8.2 | Consultar histórico após abertura rejeitada | Nenhum bilhete adicional |
| AC8.3 | Abrir após encerramento do anterior | Permitir |
| AC8.4 | Abrir após cancelamento do anterior | Permitir |
| AC8.5 | Duas aberturas simultâneas para uma placa livre | Apenas uma aceita; somente um bilhete aberto |

## 11. Decisões de negócio para lacunas do enunciado

Estas definições são escolhas do projeto para pontos não totalmente
esclarecidos pelo enunciado.

### 11.1. Quantidade no relatório

A quantidade de bilhetes corresponde aos encerrados na data consultada,
incluindo os gratuitos. Abertos e cancelados ficam fora dessa contagem.

Quando não houver encerramentos, quantidade, faturamento e média
devem ser zero.

### 11.2. Minutos incompletos

Qualquer minuto incompleto deve ser arredondado para cima na duração.
Segundos excedentes não podem ser ignorados para conceder gratuidade
ou evitar a cobrança de uma nova fração.

- 10 minutos e 1 segundo: duração de 11 minutos e cobrança de R$ 3,00.
- 30 minutos e 1 segundo: duração de 31 minutos e cobrança de R$ 6,00.

### 11.3. Encerramento após cancelamento

Um bilhete cancelado não pode ser encerrado.
A tentativa deve ser rejeitada sem gerar saída ou cobrança.

### 11.4. Empate na ordenação

Quando dois bilhetes tiverem o mesmo momento de entrada,
o de maior identificador deve aparecer primeiro.
