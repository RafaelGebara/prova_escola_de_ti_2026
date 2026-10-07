
# Constituição — Zona Azul Digital

## 1. Escopo e contrato

O projeto deve fornecer somente uma API REST de estacionamento rotativo.
Não deve incluir interface gráfica ou back-office.

A implementação deve respeitar os endpoints, campos JSON, códigos HTTP
e mensagens de erro definidos no contrato do enunciado.
Exemplos ilustrativos que contrariem o contrato não devem ser seguidos.

## 2. Parâmetros obrigatórios

- TARIFA_HORA_CENTAVOS: 600.
- FRACAO_MINUTOS: 30.
- TETO_DIARIO_CENTAVOS: 5000.
- TOLERANCIA_MINUTOS: 10.
- PORTA_SERVICO: 8005.

A API deve estar disponível em http://localhost:8005.

## 3. Integridade dos valores e cobrança

Todos os valores monetários devem ser calculados, armazenados e
retornados em centavos inteiros, sem ponto flutuante.

Cada fração de 30 minutos custa 300 centavos.
A quantidade de frações deve ser arredondada para cima.

Durações de até 10 minutos, inclusive, são gratuitas.
Ultrapassada a tolerância, a cobrança considera toda a duração,
sem descontar os 10 minutos iniciais.

> [!WARNING]
> A cobrança nunca pode ultrapassar 5000 centavos por bilhete,
> mesmo que a permanência atravesse dias. O teto não é renovado
> nem multiplicado pela quantidade de dias.

## 4. Integridade dos bilhetes

Uma placa pode ter, no máximo, um bilhete aberto por vez.
Essa restrição deve ser preservada inclusive em requisições simultâneas.

Somente bilhetes abertos podem ser encerrados ou cancelados.
O cancelamento não gera saída nem valor de cobrança.
Após encerramento ou cancelamento, a placa pode abrir outro bilhete.

Operações rejeitadas não devem alterar os dados dos bilhetes.

## 5. Datas e arredondamento

As datas e horas retornadas devem usar ISO-8601 com fuso -03:00.
A entrada opcional deve aceitar ISO-8601 com fuso e preservar
o instante informado.

O tempo médio do relatório deve considerar somente bilhetes
encerrados no dia e arredondar empates de 0,5 para cima.
Não utilizar arredondamento bancário para essa média.

## 6. Qualidade e TDD

O desenvolvimento deve seguir o ciclo Red, Green e Refactor:
primeiro escrever e executar um teste que falha; depois implementar
o necessário para passar; por fim refatorar mantendo os testes passando.

Cada caso de uso deve possuir critérios de aceite verificáveis.
Os testes devem cobrir limites de cobrança, erros e mudanças de estado.

Testes de tempo devem usar relógio controlável, sem esperas reais.
A lógica de cobrança deve ficar separada da camada HTTP.

## 7. Consistência da documentação

Os arquivos spec.md, plan.md, tests.md e tasks.md devem respeitar
esta constituição e manter os mesmos parâmetros e regras.

Decisões sobre comportamentos não definidos no enunciado devem
ser identificadas explicitamente como decisões do projeto.

Caso sejam necessários snippets de código, cada bloco deve ter no máximo 20 linhas. Não envolver o documento inteiro em um bloco de código.
