# Tarefas — Zona Azul Digital

## Diretrizes

Seguir constitution.md, spec.md, plan.md e tests.md.

Usar tarifa de 600 centavos por hora, fração de 30 minutos, teto de 5000 centavos, tolerância de 10 minutos e porta 8005.

Armazenar os bilhetes em memória durante a execução, com um único worker e sem recarga automática.

Para cada comportamento, seguir TDD: escrever e executar um teste que falha, implementar o necessário e refatorar mantendo os testes passando.

Não alterar os arquivos protegidos da prova.

## T01 — Preparar o projeto

Criar a estrutura da aplicação, serviços, cálculo, repositório e testes conforme o plano.

Declarar versões compatíveis das dependências e configurar pytest e Ruff.

Centralizar os parâmetros da variante.

Preparar a aplicação para receber repositório e relógio substituíveis nos testes.

Conclusão: estrutura importável, dependências instaláveis e ferramentas configuradas.

## T02 — Implementar armazenamento em memória

Dependência: T01.

Escrever testes para armazenamento, consulta, identificadores únicos e isolamento entre testes.

Implementar um dicionário de bilhetes, contador crescente iniciado em 1 e lock compartilhado.

Preservar bilhetes encerrados e cancelados para histórico e relatórios.

Retornar cópias consistentes dos registros, evitando alterações externas.

Conclusão: bilhetes disponíveis entre requisições da mesma execução e identificadores sem repetição.

## T03 — Implementar duração e cobrança

Dependência: T01.

Escrever testes para tolerância, frações, segundos excedentes e teto.

Cobrir durações de 0, 10, 11, 30, 31, 60, 61, 95, 480 e 481 minutos, além de 48 horas.

Implementar os cálculos com centavos inteiros e sem ponto flutuante.

Aplicar o arredondamento de minutos incompletos definido no spec.md.

Manter o cálculo independente de HTTP e do repositório.

Conclusão: valores corretos, tolerância sem desconto e cobrança limitada a 5000 centavos por bilhete.

## T04 — Implementar abertura de bilhete

Dependência: T02.

Cobertura: UC1 e UC8.

Escrever testes para abertura válida, entrada opcional, placa inválida e entrada inválida.

Testar a precedência das validações: placa, entrada e conflito de placa ocupada.

Implementar POST /bilhetes conforme o contrato.

Usar o relógio quando a entrada estiver ausente.

Proteger com o mesmo lock a verificação da placa, geração do identificador e criação do bilhete.

Testar duas aberturas simultâneas para a mesma placa.

Conclusão: abertura válida retorna 201, formato inválido retorna 422 e placa ocupada retorna 409. Solicitações simultâneas criam somente um bilhete aberto.

## T05 — Implementar encerramento

Dependências: T03 e T04.

Cobertura: UC2 e UC7.

Escrever testes para encerramento válido, bilhete inexistente e encerramento repetido.

Verificar tolerância, frações e teto também pelos testes HTTP.

Implementar POST /bilhetes/{id}/encerramento sem exigir corpo.

Capturar a saída uma única vez, calcular a cobrança e alterar o estado sob proteção do lock.

Preservar os dados originais quando uma nova tentativa de encerramento for rejeitada.

Conclusão: resposta contém id, placa, entrada, saida, minutos e valor_centavos, com tipos e erros corretos.

## T06 — Implementar cancelamento

Dependências: T04 e T05.

Cobertura: UC5 e UC8.

Escrever testes para cancelamento de bilhete aberto, inexistente, encerrado e já cancelado.

Implementar POST /bilhetes/{id}/cancelamento sem exigir corpo.

Garantir que cancelamento não gere saída nem cobrança.

Testar tentativa de encerrar bilhete cancelado.

Testar nova abertura da mesma placa após encerramento e após cancelamento.

Testar encerramento e cancelamento simultâneos, permitindo somente uma transição.

Conclusão: cancelamento preserva histórico, libera a placa e impede transições posteriores indevidas.

## T07 — Implementar ativos e histórico

Dependências: T04, T05 e T06.

Cobertura: UC3 e UC6.

Escrever testes para listas vazias, filtros e ordenação.

Implementar GET /bilhetes/ativos e GET /bilhetes?placa=ABC1D23.

Validar a placa obrigatória no histórico.

Ordenar por entrada decrescente e, em empate, identificador decrescente.

Retornar os campos correspondentes ao estado de cada bilhete.

Conclusão: ativos apresenta somente abertos e histórico apresenta todos os bilhetes da placa consultada.

## T08 — Implementar relatório diário

Dependências: T05 e T06.

Cobertura: UC4.

Escrever testes para data inválida, ausência de encerramentos, bilhetes gratuitos e exclusão de abertos e cancelados.

Testar a média de 30 e 31 minutos, esperando 31.

Testar encerramento em dia diferente da entrada e limites de meia-noite no fuso -03:00.

Implementar GET /relatorios/diario conforme o contrato e as decisões do spec.md.

Conclusão: quantidade, faturamento e média refletem os encerramentos do dia. Sem encerramentos, as três métricas são zero.

## T09 — Verificar o contrato completo

Dependências: T04 a T08.

Executar testes HTTP de todos os endpoints.

Conferir campos, tipos, códigos HTTP, mensagens de erro e fuso das datas.

Garantir que erros previstos retornem somente o campo erro, sem detail.

Verificar que operações rejeitadas não alteram os registros.

Garantir que a cobrança utilize valor_centavos, nunca o campo valor do exemplo inconsistente.

Para cada falha encontrada, escrever um teste que a reproduza antes da correção.

Conclusão: critérios de aceite cobertos e testes passando em conjunto.

## T10 — Preparar o container

Dependência: T09.

Criar Containerfile com versão explícita da imagem Python e dependências declaradas.

Executar com usuário sem privilégios de administrador.

Configurar o servidor em 0.0.0.0:8005, com um worker e sem recarga automática.

Não exigir banco externo, volumes de dados ou variáveis de ambiente.

Construir e executar o container publicando a porta 8005 para 8005.

Verificar abertura e consulta de bilhete no container.

Conclusão: API acessível em http://localhost:8005 sem preparação manual.

## T11 — Documentar e verificar a entrega

Dependência: T10.

Documentar no README a instalação, execução local, container, testes e linter.

Registrar parâmetros, endpoints e decisões sobre lacunas do contrato.

Explicar a perda dos dados ao reiniciar e a necessidade de um único worker.

Verificar ausência de segredos, caches, ambientes virtuais e arquivos temporários no versionamento.

Executar a suíte completa e o Ruff.

Conferir a presença de Containerfile, manifesto de dependências, README e testes próprios.

Conclusão: aplicação reproduzível, documentação consistente e verificações finais aprovadas.
