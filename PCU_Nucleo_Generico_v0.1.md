# PCU — Núcleo Genérico Executável v0.1
**Compatível com Constituição PCU v1.0 · independente de stack · alvo: qualquer pessoa/equipe com poucos recursos**

Convenção deste documento: cada bloco define o modelo **genérico** primeiro (vale para qualquer implementação), depois mostra como realizá-lo em duas representações mínimas — **tabela SQL** (Postgres/Supabase/SQLite, mesma estrutura) e **arquivos Markdown** (zero banco). As duas são intercambiáveis; nenhuma é "a certa".

---

## 1. Catálogo de Capacidades

### Modelo mínimo (8 campos, tecnologia-agnóstico)

| Campo | Obrigatório | Descrição |
|---|---|---|
| `id` | sim | slug único, ex.: `revisar-seguranca-sql` |
| `nome` | sim | nome curto e humano |
| `descricao` | sim | 1-2 frases: o que faz |
| `gatilhos` | sim | exemplos de quando chamar (texto livre) |
| `responsavel` | sim | quem executa — nome do agente, script, função; não precisa ser um "agente de IA" |
| `custo_estimado` | não | unidade abstrata (tokens, segundos, chamadas de API) — a unidade é escolha de quem implementa |
| `status` | sim | `ativa` / `rascunho` / `obsoleta` |
| `corpo` | sim | instruções completas de execução — **carregado só quando a capacidade é escolhida**, nunca antes |

A separação entre metadados (7 campos leves, sempre visíveis) e `corpo` (pesado, sob demanda) é o ponto central — ver Progressive Disclosure abaixo.

### Como um Agente declara Capacidades
Um Agente é apenas um agrupamento lógico de Capacidades — não existe Agente sem pelo menos 1 Capacidade declarada com `responsavel` apontando para ele. Declarar = adicionar um registro (linha ou arquivo) com os 8 campos. Nenhuma capacidade nasce implícita.

### Como o Orquestrador descobre e escolhe (com Fast-Path)
1. Orquestrador recebe uma demanda.
2. Busca **só nos metadados** (nome, descrição, gatilhos) de todas as Capacidades `ativa` — busca por palavra-chave é suficiente para o núcleo; busca semântica é opcional (Bloco 5).
3. **0 correspondências** → não existe Capacidade para isso hoje; registrar o gap, não inventar.
4. **1 correspondência clara, sem ambiguidade** → Fast-Path: carrega o `corpo` só agora, executa.
5. **2+ correspondências, ambiguidade, ou cruzamento de domínios** → Full Cycle: aciona planejamento (Constituição, Parte IV) antes de carregar qualquer `corpo`.

### Progressive Disclosure (inspirado no Hermes, sem a complexidade toda)
Regra única: **o custo de "saber que uma Capacidade existe" deve ser ordens de grandeza menor que o custo de executá-la.** Concretamente: metadados de todas as Capacidades cabem sempre no contexto do Orquestrador; o `corpo` de cada uma só entra quando ela é a escolhida. Isto é o que o Hermes chama de skill "always visible, ~100 word description" vs. corpo completo sob demanda — não precisa do resto do mecanismo do Hermes (learning loop automático, DSPy) para copiar só essa regra.

### Duas representações equivalentes

**SQL (Postgres/SQLite — mesma estrutura em qualquer um dos dois):**
```sql
CREATE TABLE capabilities (
  id text PRIMARY KEY,
  nome text NOT NULL,
  descricao text NOT NULL,
  gatilhos text NOT NULL,
  responsavel text NOT NULL,
  custo_estimado int,
  status text NOT NULL DEFAULT 'ativa' CHECK (status IN ('ativa','rascunho','obsoleta')),
  corpo text NOT NULL
);
```

**Arquivos Markdown (zero banco — uma pasta `capabilities/`, um arquivo por capacidade):**
```markdown
---
id: revisar-seguranca-sql
nome: Revisor de Segurança SQL
gatilhos: "SQL injection; query direta em produção; nova tabela"
responsavel: revisor-codigo
custo_estimado: 600
status: ativa
---
[corpo completo aqui — as instruções de verdade]
```
O frontmatter YAML é os 7 metadados leves (o que o Orquestrador lê sempre); tudo abaixo do `---` é o `corpo` (lido só quando escolhido). Ler só o frontmatter de N arquivos é uma operação de sistema de arquivos, não custa nada em tokens até a leitura do corpo acontecer.

---

## 2. Orquestração e Métricas Mínimas

### Regras operacionais do Orquestrador (papel, não implementação)
1. Nunca executa sem antes consultar o Catálogo (Bloco 1).
2. Prefere Fast-Path por padrão; Full Cycle exige justificativa registrada (Constituição, Parte III).
3. Interrompe ou degrada se o custo estimado acumulado da demanda ultrapassar um orçamento definido — o orçamento é configuração, não parte do papel.
4. Registra toda execução no log mínimo (abaixo), sem exceção.

### Fast-Path vs Full Cycle — critérios objetivos
| Critério | Fast-Path | Full Cycle |
|---|---|---|
| Nº de Capacidades necessárias | 1 | 2+ |
| Ambiguidade na intenção | nenhuma | existe |
| Cruza domínios/contextos | não | sim |
| Reversibilidade da ação | reversível/baixo risco | irreversível ou alto impacto |
| Cria Capacidade/Agente novo | não | sim |

Basta **um** critério cair na coluna Full Cycle para a demanda deixar de ser Fast-Path.

### Log mínimo de execução (uma linha por demanda, qualquer stack)
```
timestamp | demanda_resumo | capacidades_usadas | fast_path (bool) | custo_estimado | sucesso (bool) | justificativa_full_cycle (nullable)
```
Em SQL: uma tabela. Em arquivos: um `.jsonl` (uma linha JSON por execução, append-only — não precisa de banco para começar).

### As 5 métricas obrigatórias, todas derivadas só desse log
1. **% Fast-Path vs Full Cycle** = `count(fast_path=true) / count(*)`
2. **Nº médio/mediano de Capacidades por demanda** = `avg/median(len(capacidades_usadas))`
3. **Custo cognitivo por demanda** = `sum(custo_estimado)` no período
4. **Taxa de reutilização** = `capacidades usadas em 2+ demandas / total de capacidades ativas`
5. **Full Cycle sem justificativa** = `count(fast_path=false AND justificativa_full_cycle IS NULL)` — deve ser sempre 0; se não for, é violação registrada (Constituição, Parte III)

### Utilização real (evitar Capacidade/Agente parado)
`ultimo_uso` de cada Capacidade = `max(timestamp)` no log onde ela aparece em `capacidades_usadas`. Regra: Capacidade sem uso há mais de N dias (configurável, sugestão: 60) é **marcada para revisão**, nunca apagada automaticamente (Constituição P10 — evolução reversível). Se todas as Capacidades de um Agente estiverem nesse estado, o Agente inteiro é marcado para revisão.

---

## 3. Aprendizado Mínimo Viável

### O ciclo mínimo
```
Execução (log) → rascunho de Capacidade (status='rascunho') → revisão humana → 
  aceita: status='ativa' (nova ou versão atualizada de existente)
  edita: ajusta corpo antes de aceitar
  descarta: apagado, nada entra no Catálogo
```

### O que copiar do Hermes, sem a complexidade toda
- **Gatilho de geração do rascunho:** só depois de uma execução Full Cycle bem-sucedida, ou de um Fast-Path que se repetiu 3+ vezes de forma parecida (sinal de que merece virar Capacidade própria em vez de improviso repetido). Não gerar rascunho a cada Fast-Path trivial — isso é ruído, não aprendizado.
- **Formato do rascunho = mesmo formato de uma Capacidade normal** (Bloco 1), só que com `status='rascunho'` — não é um mecanismo separado, é o mesmo Catálogo com um status a mais.
- **Revisão sempre humana antes de `ativa`** — o próprio Hermes não deixa o agente instalar sozinho a partir do Hub; aqui vale o mesmo princípio: o agente/Orquestrador nunca promove sozinho um rascunho a ativo.

### O que fica fora do MVP (explicitamente)
- Otimização evolutiva de prompt/skill via traces (DSPy/GEPA) — exige infraestrutura de avaliação que não faz parte do núcleo.
- Reescrita automática de Capacidades existentes sem revisão humana.
- Busca semântica/embeddings no Catálogo — Bloco 5 trata isso como opcional, fase 2.
- Qualquer mecanismo de "auto-instalação" de Capacidade vinda de um catálogo externo sem revisão.

---

## 4. Organização de Agentes (inspirada no contains-studio)

### Organização por domínio, não por tecnologia
Agentes agrupam Capacidades por **domínio de negócio ou capacidade horizontal reconhecível** (ex.: "atendimento ao cliente", "marketing", "revisão de código") — nunca por acidente técnico (ex.: "agente que usa a API X"). O teste: um agente deve fazer sentido para alguém de fora da equipe técnica ler o nome e entender o que ele cobre.

### Regra: criar Agente novo vs. estender Capacidade existente
- **Estender:** a nova competência cabe no domínio de um Agente existente e não muda seu perfil de risco/acesso a dados → adicionar Capacidade a ele.
- **Criar novo:** a competência abre um domínio de negócio não coberto por nenhum Agente atual, **ou** exige isolamento (acesso a dados diferente, risco diferente) que não deveria ficar misturado com o que já existe.
- Pergunta obrigatória antes de criar: "isto seria útil considerando o conjunto todo, não só o caso que motivou o pedido?" Se não, é Capacidade de um Agente existente, não Agente novo.

### Evitar agentes mortos-vivos
Mesma regra de utilização do Bloco 2, aplicada no nível do Agente: se todas as suas Capacidades estão sem uso além do limiar configurado, o Agente é marcado para revisão — arquivar, fundir com outro, ou reativar conscientemente. Nunca fica "existindo" silenciosamente sem ninguém saber se ainda serve.

---

## 5. Adequação a Baixo Recurso

### Obrigatório no núcleo (custo ~zero)
- Catálogo de Capacidades — mesmo como pasta de arquivos Markdown, sem banco
- Papel de Orquestrador — pode ser o próprio humano ou um único LLM barato seguindo a regra do Bloco 2, sem componente separado
- Log mínimo de execução — mesmo que um `.jsonl` local
- Aplicação da regra Fast-Path/Full Cycle — mesmo que decidida manualmente a cada vez, sem automação

### Opcional / fase 2 (só adicionar quando o núcleo já estiver rodando)
- Busca semântica/embeddings no Catálogo (fase 1: busca por palavra-chave já resolve)
- Dashboard de métricas (fase 1: consulta manual ao log quando precisar)
- Geração automática de rascunho de Capacidade (fase 1: revisão 100% manual, sem gatilho automático)
- Coordenação multiagente tipo swarm, federação entre máquinas — nada disto faz parte do núcleo em nenhuma fase próxima

### Caminho de bootstrap — 1 modelo barato + SQLite ou arquivos, zero infraestrutura
1. Criar pasta `capabilities/`, um `.md` por Capacidade (frontmatter + corpo), começando com 3-5 capacidades reais que você já faz manualmente.
2. Criar um arquivo `log.jsonl`, append-only, uma linha por execução.
3. O "Orquestrador" é o próprio modelo, com um prompt fixo curto: "leia o frontmatter de todos os arquivos em `capabilities/`, escolha o que resolve o pedido, carregue só o corpo do escolhido, execute, registre uma linha no log."
4. Sem SQL, sem servidor. Migrar para SQLite quando o número de arquivos tornar a busca manual lenta; para Postgres/Supabase só quando precisar de concorrência real (múltiplos usuários/processos escrevendo ao mesmo tempo).

---

## 6. Próximos 3 Passos Concretos

1. **Formalizar o esquema de Capacidade como especificação escrita** (frontmatter YAML ↔ tabela SQL, os mesmos 8 campos nos dois formatos) com um exemplo mínimo funcionando nas duas representações lado a lado — hoje isso só existe amarrado a uma implementação SQL específica; sem essa formalização, "genérico" continua sendo promessa.
2. **Implementar o log mínimo de execução e as 5 métricas do Bloco 2** em qualquer stack de teste — sem isso, Fast-Path vs Full Cycle continua sendo regra no papel, não medição real.
3. **Rodar o ciclo de aprendizado mínimo (Bloco 3) de ponta a ponta em 1 caso real único** antes de generalizar para todas as Capacidades — provar que rascunho → revisão → ativa funciona uma vez, documentar o resultado, só então formalizar como processo padrão.
