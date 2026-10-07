# Orquestrador — Zona Azul Digital

## 1. Finalidade

Orientar o professor e o modelo responsável pela geração da API sobre a ordem de leitura e aplicação dos documentos desta entrega.

Os arquivos Markdown são especificações: devem ser lidos em conjunto antes de gerar código. A ordem de leitura não significa implementar cada documento isoladamente.

## 2. Ordem de leitura

| Ordem | Arquivo | Finalidade |
| --- | --- | --- |
| 1 | constitution.md | Conhecer os princípios, restrições e convenções do projeto |
| 2 | spec.md | Entender as regras de negócio, os oito casos de uso e os critérios de aceite |
| 3 | plan.md | Conhecer as decisões técnicas, o contrato HTTP e a estratégia de implementação |
| 4 | tasks.md | Identificar a decomposição do trabalho e as dependências entre tarefas |
| 5 | tests.md | Conhecer os cenários, os casos de borda e os resultados esperados antes de implementar |

Consultar também ENUNCIADO.md e contrato.json, que estabelecem os requisitos obrigatórios da prova. Os parâmetros aplicáveis são os da variante deste repositório.

## 3. Conferência antes da implementação

Verificar a consistência entre os cinco documentos e o contrato obrigatório.

Utilizar os seguintes parâmetros:

| Parâmetro | Valor |
| --- | ---: |
| TARIFA_HORA_CENTAVOS | 600 |
| FRACAO_MINUTOS | 30 |
| TETO_DIARIO_CENTAVOS | 5000 |
| TOLERANCIA_MINUTOS | 10 |
| PORTA_SERVICO | 8005 |

Seguir a solução aprovada no plan.md: armazenamento em memória, execução com um único worker, sem recarga automática e serviço na porta 8005. Os dados permanecem disponíveis entre requisições da mesma execução e são perdidos ao reiniciar.

Não adicionar banco de dados, interface gráfica ou outros componentes fora do escopo definido.

## 4. Ordem de execução das tarefas

Após a leitura completa, executar as tarefas na seguinte sequência, respeitando as dependências detalhadas em tasks.md:

1. T01 — Preparar o projeto.
2. T02 — Implementar armazenamento em memória.
3. T03 — Implementar duração e cobrança.
4. T04 — Implementar abertura de bilhete.
5. T05 — Implementar encerramento.
6. T06 — Implementar cancelamento.
7. T07 — Implementar ativos e histórico.
8. T08 — Implementar relatório diário.
9. T09 — Verificar o contrato completo.
10. T10 — Preparar o container.
11. T11 — Documentar e verificar a entrega.

A preparação estrutural vem primeiro. Para cada tarefa que acrescenta comportamento, aplicar o ciclo TDD descrito a seguir.

## 5. Aplicação do TDD

Antes de implementar cada comportamento:

1. Consultar a regra e os critérios de aceite correspondentes no spec.md.
2. Consultar o contrato e as decisões técnicas no plan.md.
3. Selecionar os cenários correspondentes em tests.md.
4. Escrever os testes automatizados e executá-los, confirmando a falha pelo comportamento ainda ausente — Red.
5. Implementar o necessário para os testes passarem — Green.
6. Refatorar preservando o comportamento e executar novamente os testes — Refactor.
7. Verificar os critérios de conclusão da tarefa antes de avançar.

Não deixar a escrita dos testes para depois de toda a implementação. A posição de tests.md na ordem de leitura não o torna a última etapa de desenvolvimento.

Utilizar relógio controlável e repositório isolado nos testes, conforme o plano. Não aguardar tempo real para verificar tolerância, frações ou teto.

## 6. Tratamento de divergências

Os requisitos explícitos do contrato obrigatório prevalecem sobre exemplos ilustrativos e escolhas da entrega que os contrariem.

As decisões identificadas como lacunas no spec.md e no plan.md complementam comportamentos não definidos pelo contrato. Não devem ser confundidas com exigências originais da prova.

A divergência entre porta_interna 8080 e a orientação textual sobre a porta da variante já foi tratada no plano: utilizar 8005 para o processo e sua exposição no container.

Este orquestrador organiza a execução; não substitui nem acrescenta regras de negócio. Se houver outra contradição sem resolução documentada, identificá-la explicitamente antes de implementar o comportamento afetado.

## 7. Verificação final

Concluir a geração somente após verificar:

- Atendimento aos oito casos de uso e aos critérios de aceite.
- Conformidade de rotas, campos, tipos, códigos HTTP e mensagens de erro.
- Execução bem-sucedida dos testes especificados e do linter.
- Container acessível em http://localhost:8005, sem configuração manual obrigatória.
- Presença de Containerfile, manifesto de dependências, README e testes próprios.
- Documentação do armazenamento volátil e da execução com um único worker.
- Preservação dos arquivos protegidos da prova.

Este documento não altera o procedimento de entrega do aluno, a auto-correção ou as regras de avaliação estabelecidas pelo professor.
