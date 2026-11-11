# Bot de Clima no Telegram com n8n

Workflow que recebe uma cidade brasileira pelo Telegram, consulta a OpenWeather e responde com a temperatura atual arredondada em graus Celsius.

Repositório: [telegram-weather-FTR](https://github.com/sostenesfirmino/telegram-weather-FTR).

## Arquivos

- `workflow-chatbot-telegram.json`: workflow exportado do n8n e preparado para importação, sem credenciais ou tokens embutidos.
- `README.md`: instruções para configurar e executar o projeto.

Versão utilizada no desenvolvimento: **n8n 2.47.3**, executado em Docker. Não é necessário Gemini nem outro serviço de IA.

## Pré-requisitos

- Uma instância do n8n com acesso à internet e permissão para ler a variável de ambiente indicada abaixo.
- Um bot criado pelo [BotFather](https://t.me/BotFather) e seu token.
- Uma conta na OpenWeather e uma chave válida de [API Keys](https://home.openweathermap.org/api_keys). A API Current Weather está disponível no plano gratuito.
- Um endereço HTTPS público para os webhooks do Telegram. No desenvolvimento foi utilizado ngrok; o editor do n8n pode permanecer acessível na rede local.

## Variáveis e credenciais

| Nome esperado | Configuração | Finalidade |
| --- | --- | --- |
| `OPENWEATHER_API_KEY` | Variável de ambiente do processo/contêiner que executa o n8n | Autenticar a consulta à OpenWeather. |
| `TELEGRAM_BOT_TOKEN` | Token obtido no BotFather, inserido no campo **Access Token** de uma credencial **Telegram API** | Autenticar os nós Telegram. |

`TELEGRAM_BOT_TOKEN` é o nome documentado para o segredo do bot. O workflow utiliza a credencial nativa Telegram API: definir apenas essa variável no ambiente não cria nem vincula a credencial automaticamente.

Configure `OPENWEATHER_API_KEY` no ambiente da sua instalação antes de executá-la. Em Docker, forneça a variável ao contêiner pelo mecanismo usado para iniciá-lo. Para mudar o ambiente de um contêiner existente, recrie-o preservando seus volumes e configurações; apenas reiniciar não injeta novas variáveis. Não é obrigatório utilizar Docker Compose.

Se o acesso a `$env` estiver bloqueado, o administrador deve permitir esse acesso pela configuração `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`. Essa permissão permite aos autores de workflows consultar variáveis do ambiente e deve ser considerada no contexto da instância.

Para o n8n 2.47.3 atrás de um túnel/proxy, configure `N8N_WEBHOOK_URL` com a URL HTTPS pública, terminada em `/`, e ajuste `N8N_PROXY_HOPS` à quantidade real de proxies confiáveis. O túnel precisa encaminhar POST para `/webhook/` e `/webhook-test/`. Não é necessário publicar o editor.

Não inclua valores reais de chaves em arquivos versionados. Todos os segredos devem ser configurados no ambiente ou no armazenamento de credenciais do n8n.

## Importar e configurar

1. Baixe `workflow-chatbot-telegram.json` deste repositório.
2. No editor do n8n, crie um workflow e escolha **Import from File** no menu.
3. Selecione o arquivo JSON.
4. Abra **Telegram Trigger** e crie uma credencial **Telegram API**. Cole seu `TELEGRAM_BOT_TOKEN` no campo **Access Token** e salve.
5. Selecione a mesma credencial nos nós **Enviar temperatura**, **Enviar erro de cidade** e **Enviar erro do serviço**.
6. Confirme que a variável `OPENWEATHER_API_KEY` está disponível ao n8n.
7. No nó **Consultar OpenWeather**, mantenha **Authentication: None** e **Send Query Parameters** ativado.
8. Confira os parâmetros abaixo. O valor de `appid` deve estar no modo **Expression**, sem substituir a expressão por uma chave literal.

Endpoint: `GET https://api.openweathermap.org/data/2.5/weather`.

| Parâmetro | Valor |
| --- | --- |
| `q` | `{{ $json.queue }}` |
| `units` | `metric` |
| `lang` | `pt_br` |
| `appid` | `{{ $env.OPENWEATHER_API_KEY }}` |

No JSON exportado, as expressões possuem o prefixo `=`, por exemplo: `={{ $env.OPENWEATHER_API_KEY }}`.

O enunciado chama o parâmetro HTTP de `queue`, mas o nome aceito pela OpenWeather é **`q`**. A variável interna exigida continua se chamando **`queue`** e seu valor é enviado em `q`.

## Como executar

1. Com o workflow desativado/despublicado, clique em **Execute workflow** para iniciar a escuta de teste.
2. No Telegram, abra uma conversa privada com seu bot.
3. Envie `Recife,PE,BR`.
4. Aguarde uma resposta no formato abaixo; a temperatura é ilustrativa:

> 🌤️ A temperatura em Recife é de 28°C.

Se a escuta encerrar após o evento, inicie uma nova execução para a próxima mensagem. Ao terminar os testes, publique/ative o workflow e faça uma nova consulta para verificar o webhook de produção.

O Telegram registra apenas um webhook por bot. Não use o mesmo bot em outro workflow ativo enquanto testa, pois o registro de um webhook substitui o anterior.

## Funcionamento do fluxo

1. **Telegram Trigger** recebe mensagens; o IF seguinte aceita apenas texto não vazio.
2. **Preparar entrada**, um Set/Edit Fields, cria `queue` e preserva o ID do chat. Remove acentos, converte para minúsculas, remove espaços nas extremidades e junto às vírgulas, e reduz espaços repetidos. Preserva espaços entre palavras da cidade.
3. **Entrada válida?** valida o formato, a UF brasileira e o país BR. Também aceita `Cidade,UF`, acrescentando `,br`.
4. **Consultar OpenWeather** faz a requisição com timeout de 15 segundos, retornando corpo e status HTTP para validação.
5. **Resposta válida?** exige HTTP 200, `cod` 200, país BR, nome não vazio e temperatura numérica finita.
6. **Formatar temperatura** usa `Math.round()` e prepara a mensagem amigável.
7. Os nós Telegram enviam o resultado ao chat de origem ou a mensagem de erro correspondente.

Exemplo de normalização: ` São Paulo , SP , BR ` → `sao paulo,sp,br`.

Para entrada inválida, cidade não encontrada (HTTP 404) ou dados inesperados em uma resposta HTTP 200:

> ❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).

Para falhas técnicas como HTTP 401, 429, 5xx ou timeout:

> ⚠️ Não consegui consultar o clima agora. Tente novamente em alguns instantes.

Fotos, áudios e mensagens sem texto são ignorados. `/start` recebe a orientação de formato. Temperaturas de 0°C e negativas são válidas.

## Testes

| Entrada | Resultado esperado |
| --- | --- |
| `Recife,PE,BR` | Nome da cidade e temperatura atual arredondada em °C. |
| `São Paulo,SP,BR` | Nome da cidade e temperatura atual arredondada em °C. |
| `Belo Horizonte,MG,BR` | Nome da cidade e temperatura atual arredondada em °C. |
| `CidadeInexistenteXYZ,PE,BR` | Mensagem de cidade não encontrada. |
| ` São Paulo , SP , BR ` | Normalização e consulta da cidade. |
| `Recife,XX,BR` | Orientação de formato, sem chamada à API. |

Foi confirmado um teste real completo com **Recife**, recebendo a mensagem de 28°C, e o retorno da orientação para `/start`. A integração real utilizou temporariamente uma credencial **Query Auth**, armazenada no n8n. Para entrega, a autenticação foi substituída pela expressão de ambiente exigida no desafio; essa modalidade ainda precisa ser validada em um ambiente com a variável configurada.

Também foram realizadas 29 verificações locais com dados simulados na versão inicial, cobrindo expressões, conexões, cidades, entradas inválidas, temperaturas zero/negativas e falhas de API. Elas não substituem os testes reais das demais cidades e da cidade inexistente, que permanecem pendentes de confirmação.

## Limites e solução de problemas

- **HTTP 401:** verifique se a chave da OpenWeather é válida e está ativada. Chaves novas podem levar até duas horas para ativação após o cadastro.
- **Webhook sem receber mensagens:** confira URL HTTPS, túnel ativo, encaminhamento dos caminhos e ausência de outro workflow utilizando o mesmo bot.
- **Variável inacessível:** confira sua configuração no ambiente e a permissão de acesso a `$env`. O aviso da prévia do editor não substitui um teste de execução.
- **Municípios homônimos:** a documentação da OpenWeather limita a busca por estado a localidades dos EUA. O fluxo envia `cidade,uf,br` conforme o desafio e exige país BR na resposta, mas não garante a desambiguação de UFs brasileiras. Uma garantia por município exigiria resolver cidade/UF em coordenadas antes da consulta.
- A busca por nome é uma funcionalidade legada da OpenWeather, ainda disponível; o reconhecimento dos nomes depende do catálogo do provedor.

## Segurança da entrega

O JSON não contém chaves reais, tokens, referências locais de credenciais nem dados fixados de mensagens. As credenciais Telegram devem ser selecionadas após a importação. Metadados da instância de origem foram removidos.

O workflow é entregue desativado. A persistência de dados de execuções está desabilitada nas configurações exportadas. Não publique arquivos `.env`, exportações de credenciais, bancos de dados ou volumes Docker.

## Referências

- [OpenWeather — Current Weather](https://openweathermap.org/api/current)
- [OpenWeather — FAQ e erros de autenticação](https://openweathermap.org/faq)
- [n8n — HTTP Request](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/)
- [n8n — credenciais Telegram](https://docs.n8n.io/integrations/builtin/credentials/telegram/)
- [n8n — configuração de webhooks com proxy](https://docs.n8n.io/hosting/configuration/configuration-examples/webhook-url/)
