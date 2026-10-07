# Plano técnico — Zona Azul Digital

## 1. Objetivo

Orientar a implementação da API descrita em spec.md, respeitando
constitution.md e o contrato obrigatório da prova.

A entrega gerada deve incluir aplicação, testes automatizados,
Containerfile, manifesto de dependências e README de execução.

Não implementar interface gráfica, autenticação ou funcionalidades
adicionais que não sejam necessárias ao escopo.

## 2. Tecnologias e justificativas

| Tecnologia | Finalidade | Justificativa |
| --- | --- | --- |
| Python | Linguagem da aplicação | Permite implementar e testar as regras com pouca complexidade |
| FastAPI | API REST | Oferece roteamento, validação e documentação dos endpoints |
| Uvicorn | Servidor HTTP | Executa a aplicação FastAPI |
| Estruturas nativas do Python | Armazenamento em memória | Dicionários e listas bastam para o escopo, sem dependências externas |
| threading | Sincronização | Lock da biblioteca padrão garante operações atômicas no repositório |
| pytest | Testes automatizados | Permite organizar cenários e casos parametrizados |
| HTTPX | Testes HTTP | Permite verificar requisições e respostas da aplicação |

As versões compatíveis das dependências devem ser fixadas no
manifesto da implementação gerada e verificadas pela execução
dos testes.

## 3. Organização da aplicação

Separar as responsabilidades em quatro partes:

| Parte | Responsabilidade |
| --- | --- |
| API | Receber requisições, validar formatos e montar respostas HTTP |
| Serviços de negócio | Abrir, encerrar, cancelar e consultar bilhetes |
| Cálculo de cobrança | Calcular duração, tolerância, frações, teto e média |
| Repositório em memória | Consultar e gravar bilhetes com operações atômicas |

O cálculo de cobrança não deve depender de HTTP nem do repositório.
Isso permite testar seus limites isoladamente.

Os serviços devem receber o repositório e o relógio
como dependências substituíveis nos testes.

## 4. Configuração e inicialização

| Configuração | Valor padrão obrigatório |
| --- | ---: |
| TARIFA_HORA_CENTAVOS | 600 |
| FRACAO_MINUTOS | 30 |
| TETO_DIARIO_CENTAVOS | 5000 |
| TOLERANCIA_MINUTOS | 10 |
| PORTA_SERVICO | 8005 |

A aplicação deve iniciar sem exigir variáveis de ambiente,
arquivos secretos ou configuração manual.

O servidor deve escutar em 0.0.0.0:8005.
A URL de acesso da correção será http://localhost:8005.

Executar o servidor com um único processo (worker).
Cada processo teria seu próprio repositório em memória,
e múltiplos workers quebrariam a unicidade e o histórico.

### Decisão sobre a divergência de portas

O contrato contém porta_interna igual a 8080, mas sua observação
manda o serviço escutar na porta da variante.

Adotar 8005 para o processo e para a porta exposta pelo container,
seguindo essa observação e a URL da suíte.
Registrar essa decisão no README gerado.

## 5. Armazenamento em memória

Manter os bilhetes em um repositório compartilhado pelas requisições
durante a execução da aplicação.

Utilizar um dicionário indexado pelo identificador do bilhete
e um contador crescente para gerar identificadores únicos.
O primeiro identificador de uma instância vazia deve ser 1.

Não reutilizar identificadores de bilhetes encerrados ou cancelados.
Manter todos os bilhetes no repositório para histórico e relatórios.

Os dados são voláteis: reiniciar o processo inicia um repositório vazio.
A manutenção dos dados após reinício não faz parte desta solução.

### Dados do bilhete

| Campo | Representação | Restrição |
| --- | --- | --- |
| id | Inteiro | Gerado pelo contador do repositório |
| placa | Texto | Obrigatório |
| entrada | Instante UTC em microssegundos inteiros | Obrigatório |
| status | Texto | aberto, encerrado ou cancelado |
| saida | Instante UTC em microssegundos inteiros | Somente para encerrados |
| minutos | Inteiro | Somente para encerrados |
| valor_centavos | Inteiro | Somente para encerrados; entre 0 e 5000 |

Campos de encerramento podem ser nulos no repositório para bilhetes
abertos ou cancelados, mas devem ser omitidos nas respostas
desses bilhetes.

Manter um dicionário auxiliar de placa para o id do bilhete aberto,
usado na verificação de placa ocupada.
Consultas por placa, status e saída podem filtrar e ordenar
os bilhetes em memória, o que é suficiente para o escopo.

### Integridade e concorrência

Garantir no máximo um bilhete aberto por placa por meio do
dicionário auxiliar de placas abertas.

Proteger o repositório com um único lock (threading.Lock).
Abertura, encerramento e cancelamento devem executar a verificação
e a alteração dentro da mesma seção crítica.

Na abertura, verificar a placa no dicionário auxiliar, criar o bilhete
e registrar a placa sem liberar o lock entre esses passos.
Placa já registrada deve produzir o conflito bilhete_em_aberto,
sem criar bilhetes nem consumir identificadores.

No encerramento e cancelamento, verificar e alterar o estado
dentro da mesma seção crítica e remover a placa do dicionário auxiliar.
Requisições simultâneas não podem sobrescrever uma transição já concluída.

Manter as seções críticas curtas: validar formatos e calcular
valores que não dependem do estado fora do lock.

Retornar cópias dos bilhetes nas consultas, evitando que
as camadas externas alterem o estado armazenado.

## 6. Relógio e representação de datas

Disponibilizar uma dependência de relógio que forneça o instante atual.

Em produção, utilizar o relógio do sistema.
Nos testes, substituir por um relógio controlado.

Capturar o instante uma única vez por operação, evitando diferenças
entre o cálculo da duração e a saída registrada.

Aceitar entrada opcional em ISO-8601 com fuso explícito.
Converter o instante para UTC ao armazenar.

Nas respostas, converter datas e horas para o fuso fixo -03:00
e serializar em ISO-8601.

Rejeitar entrada sem fuso, datas impossíveis ou valores que
não sejam strings válidas.

Não criar endpoint público para alterar o relógio.
Não exigir espera real nos testes.

## 7. Cálculos

### Duração

Calcular a diferença entre saída e entrada com precisão inteira,
sem conversão intermediária para ponto flutuante.

Seguir a decisão do spec.md: qualquer minuto incompleto
é arredondado para cima no campo minutos.

### Cobrança

- Até 10 minutos: zero.
- Acima da tolerância: considerar toda a duração.
- Quantidade de frações: divisão por 30, arredondada para cima.
- Valor por fração: 300 centavos.
- Valor final: o menor entre a cobrança calculada e 5000 centavos.

> [!WARNING]
> O teto de 5000 centavos é aplicado uma única vez por bilhete,
> independentemente da quantidade de dias de permanência.

Calcular, armazenar e retornar dinheiro somente em centavos inteiros.
Não utilizar valores monetários em ponto flutuante, evitando
imprecisões de representação binária.

### Relatório

Interpretar a data consultada no fuso -03:00.
Selecionar saídas a partir da meia-noite inclusive até a
meia-noite seguinte exclusive.

Conforme a decisão de negócio do spec.md, utilizar os bilhetes
encerrados nesse intervalo para as três métricas:

- total_bilhetes: quantidade de registros.
- faturamento_centavos: soma dos valores cobrados.
- tempo_medio_minutos: média das durações registradas.

Calcular o arredondamento da média por quociente e resto inteiros:
quando o dobro do resto for maior ou igual ao divisor, incrementar
o quociente. Isso garante que 0,5 seja arredondado para cima.

Sem registros, retornar zero nas três métricas.

## 8. Contrato HTTP

Todas as respostas com conteúdo devem utilizar JSON.

### 8.1. Abrir bilhete

POST /bilhetes

Corpo: placa obrigatória e entrada opcional.

Sucesso: HTTP 201 com exatamente os campos
id, placa, entrada e status.

O status retornado deve ser aberto.

Erros:

| Situação | HTTP | erro |
| --- | ---: | --- |
| Placa ausente ou inválida | 422 | placa_invalida |
| Entrada presente e inválida | 422 | entrada_invalida |
| Placa com bilhete aberto | 409 | bilhete_em_aberto |

### 8.2. Encerrar bilhete

POST /bilhetes/{id}/encerramento

Não exigir corpo na requisição.

Sucesso: HTTP 200 com exatamente os campos
id, placa, entrada, saida, minutos e valor_centavos.

Não substituir valor_centavos por valor.

| Situação | HTTP | erro |
| --- | ---: | --- |
| Bilhete inexistente | 404 | bilhete_nao_encontrado |
| Bilhete já encerrado | 409 | bilhete_ja_encerrado |
| Bilhete cancelado | 409 | bilhete_nao_aberto |

O erro para bilhete cancelado é a tradução técnica da decisão
de negócio adotada no spec.md.

### 8.3. Listar ativos

GET /bilhetes/ativos

Sucesso: HTTP 200 com array de bilhetes abertos.
Cada item contém id, placa, entrada e status.

Ordenar por entrada decrescente e, em empate, id decrescente.
Sem bilhetes abertos, retornar array vazio.

### 8.4. Relatório diário

GET /relatorios/diario?data=AAAA-MM-DD

Sucesso: HTTP 200 com exatamente os campos
data, total_bilhetes, faturamento_centavos e tempo_medio_minutos.

Os três campos de métricas devem ser inteiros.

Data ausente, fora do formato exato ou inexistente:
HTTP 422 com erro data_invalida.

### 8.5. Cancelar bilhete

POST /bilhetes/{id}/cancelamento

Não exigir corpo na requisição.

Sucesso: HTTP 200 com id, placa, entrada e status cancelado.
Omitir saida, minutos e valor_centavos.

| Situação | HTTP | erro |
| --- | ---: | --- |
| Bilhete inexistente | 404 | bilhete_nao_encontrado |
| Bilhete encerrado ou cancelado | 409 | bilhete_nao_aberto |

### 8.6. Histórico por placa

GET /bilhetes?placa=ABC1D23

Sucesso: HTTP 200 com array contendo todos os bilhetes da placa.

Todos os itens contêm id, placa, entrada e status.
Encerrados também contêm saida, minutos e valor_centavos.

Ordenar por entrada decrescente e, em empate, id decrescente.
Placa sem histórico retorna array vazio.

Placa ausente ou inválida:
HTTP 422 com erro placa_invalida.

## 9. Validações e respostas de erro

Validar placa como string com exatamente 7 caracteres ASCII
entre A–Z e 0–9. Não converter minúsculas, remover espaços
ou aceitar outros tipos por coerção.

Na abertura, seguir esta ordem:

1. Validar placa.
2. Validar entrada, quando presente.
3. Verificar existência de bilhete aberto.
4. Criar o bilhete.

Uma entrada inválida deve resultar em 422 mesmo que a placa
já tenha bilhete aberto.

Implementar tratamento das validações do framework para os casos
previstos no contrato. Não retornar o formato padrão detail
nesses casos.

O corpo dos erros previstos deve conter somente o campo erro
com a mensagem correspondente.

Falhas de validação e conflitos devem preservar os dados existentes.

## 10. Estratégia de testes e TDD

Executar cada incremento em três etapas:

1. Red: escrever e executar o teste, confirmando falha pelo
   comportamento ainda ausente.
2. Green: implementar o necessário para fazê-lo passar.
3. Refactor: melhorar a estrutura sem alterar o comportamento,
   mantendo os testes passando.

### Níveis de teste

| Nível | Verificações |
| --- | --- |
| Unidade | Tolerância, frações, teto, duração e média |
| Integração | Repositório em memória, transições de estado e concorrência |
| HTTP | Rotas, campos, tipos, status e mensagens exatas |
| Execução | Inicialização e acesso à API no container |

Usar relógio controlado para testar durações exatas e segundos excedentes.

Criar uma nova instância do repositório em memória para cada teste.
Os testes não devem depender da ordem de execução, de serviços
externos ou de dados deixados por testes anteriores.

Verificar concorrência com duas solicitações simultâneas para a mesma
placa, usando threads, confirmando uma abertura aceita e outra rejeitada.

Os casos detalhados e resultados esperados estarão em tests.md,
com referência aos critérios de aceite do spec.md.

## 11. Container e entrega executável

Gerar Containerfile com:

- Imagem base de Python com versão explícita e compatível.
- Instalação das dependências declaradas.
- Cópia dos arquivos necessários à aplicação.
- Usuário sem privilégios de administrador.
- Porta 8005 exposta.
- Comando de inicialização do servidor em 0.0.0.0:8005,
  com um único worker.

O repositório é criado vazio a cada inicialização do container.
Não há volume nem arquivo de dados a preservar.

O container deve iniciar sem variáveis de ambiente obrigatórias.

Verificar o acesso à API com a porta 8005 do host publicada
para a porta 8005 do container.

## 12. Documentação e higiene do código gerado

O README da aplicação gerada deve apresentar:

- Tecnologias e organização do projeto.
- Parâmetros da variante.
- Instalação e execução local.
- Construção e execução do container.
- Execução dos testes e do linter.
- Armazenamento em memória e perda dos dados ao reiniciar.
- Exemplos de uso dos endpoints.
- Decisões adotadas para lacunas e divergências do contrato.

Declarar dependências de execução e de desenvolvimento.
Utilizar Ruff para verificação estática do código gerado.

Não incluir segredos, ambientes virtuais, caches
ou arquivos temporários no versionamento.

Não alterar arquivos protegidos da prova.

## 13. Critérios de conclusão técnica

A implementação gerada estará pronta quando:

- Todos os critérios de aceite do spec.md forem atendidos.
- O contrato HTTP estiver reproduzido sem mudanças de campos ou erros.
- Os testes de tests.md estiverem implementados e passando.
- O linter não apresentar erros.
- O container iniciar e responder na porta 8005.
- A aplicação funcionar sem configuração manual obrigatória.
- Os dados permanecerem consistentes e compartilhados entre
  requisições durante a execução do processo.
- O README permitir reproduzir instalação, execução e testes.
