# Chronos - Multi-Tenant AI Assistant & Task Scheduler

**Nota de Engenharia:** Este repositório documenta a arquitetura, as decisões de design e o fluxo de dados do Chronos. Por se tratar de um produto comercial ativo (SaaS) com integrações de pagamentos e dados de usuários, o código-fonte original é mantido em um repositório privado. O descritivo abaixo detalha a engenharia do sistema.

## Visão Geral
O Chronos é um assistente virtual e gerenciador de tarefas multiusuário (Multi-Tenant) que opera nativamente via WebSockets no WhatsApp. O sistema utiliza Processamento de Linguagem Natural (NLP) e integrações com LLMs para transcrição de áudio e extração de intenções temporais, permitindo que os usuários agendem lembretes complexos utilizando linguagem coloquial.

## Stack Tecnológica
* **Backend:** Node.js, JavaScript
* **Banco de Dados:** SQLite (Relacional)
* **Comunicação:** WebSockets (via biblioteca Baileys)
* **Processamento de Tempo:** date-fns, date-fns-tz
* **Integrações:** API Gemini (LLM), API de Pagamentos (Pix), Serviço de SMTP.

## Arquitetura e Soluções de Engenharia

### 1. Gerenciamento de Concorrência (Worker-Pool)
Para evitar o estrangulamento do Event Loop do Node.js durante picos de recebimento de mensagens, foi arquitetada uma fila de processamento assíncrono (`incomingQueue`) controlada por um *Worker-pool* (`tickIncoming`). 
* O sistema limita dinamicamente a concorrência baseada nos núcleos de CPU disponíveis (`os.cpus()`), garantindo que a ingestão de dados e as chamadas de banco de dados ocorram sem travamentos ou perda de pacotes de rede.

### 2. Máquina de Estados Finita (Conversational FSM)
O gerenciamento de sessões de múltiplos usuários simultâneos é feito através de uma Máquina de Estados mapeada em memória (`userStates`). 
* O sistema rastreia o contexto exato de cada usuário (ex: aguardando nome da tarefa, selecionando dias da semana, fluxo de pagamento) e intercepta as respostas roteando para os *handlers* corretos, permitindo navegação complexa de menus sem perda de estado.

### 3. Extração de Intenções Temporais e Fuso Horário (Temporal Logic)
Lidar com tempo distribuído é um dos maiores desafios de engenharia. O Chronos resolve isso através de:
* **LLM Parsing:** Usuários podem enviar textos ou áudios não estruturados (ex: "Me lembre de ir ao médico na próxima quarta de tarde"). O sistema envia o contexto temporal atual para a API do Gemini, que devolve um objeto estruturado em ISO 8601.
* **Timezone Mapping:** O banco de dados armazena o fuso horário geográfico de cada usuário, garantindo que o disparo do *cron* de envio converta as datas UTC (salvas no SQLite) para o horário local exato do destinatário via `date-fns-tz`.

### 4. Engine de Assinaturas e Pagamentos
O sistema atua como um micro-SaaS completo, gerenciando hierarquias de permissão (Comum, Pro, VIP, Vitalício).
* Implementação de geração dinâmica de cobranças via integração de API Pix.
* Execução de *Background Jobs* ("Ceifador"): Uma rotina de verificação contínua que varre o banco de dados em busca de assinaturas expiradas, executando o *downgrade* automático do plano do usuário e revogando acesso a features premium (como transcrição de áudio).

## Fluxo de Dados (Pipeline de Mensagem)
1. **Ingestão:** O socket do WhatsApp recebe o payload. A mensagem é normalizada (resolução de JIDs) e inserida na `incomingQueue`.
2. **Classificação:** O Worker retira a mensagem da fila e verifica permissões e limites de rate-limit via banco de dados relacional.
3. **Processamento (Áudio/Texto):** Caso seja áudio de um usuário VIP, o arquivo é decodificado e enviado para transcrição.
4. **Resolução de Estado ou NLP:** A mensagem é roteada pela Máquina de Estados (para comandos duros) ou enviada para o módulo de IA para interpretação de linguagem natural.
5. **Persistência e Agendamento:** Os dados extraídos são persistidos no SQLite, ativando os gatilhos de agendamento que farão o *polling* para enviar a resposta no momento adequado.
