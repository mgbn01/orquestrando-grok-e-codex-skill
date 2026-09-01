---
name: orquestrando-grok-e-codex
description: Use when work would benefit from a second AI engine on this machine — bulk read-only codebase surveys, adversarial review of a diff or security fix, or a factual matrix that needs file:line evidence. Also when the user says orquestrar, delegar, segunda opinião, revisão adversarial, grok, codex, or "effort high".
---

# Orquestrando Grok e Codex

## Overview

Claude Code é o maestro; Grok e Codex são dois motores diferentes chamados por Bash em modo headless. O valor não é velocidade — é **viés diferente**: um motor treinado de outro jeito enxerga o que o meu não enxerga.

**Princípio central: eles produzem, eu verifico e assino.** Todo achado de delegado é *hipótese*, não fato, até eu reproduzir no código.

## Divisão de papéis

| Papel | Quem | Serve para |
|---|---|---|
| **Levantamento** | Grok | Varredura factual em volume: inventariar rotas, mapear tudo que uma CSP precisa permitir, achar todos os `<script>` inline, listar TODOs, migrar padrão em N arquivos. Read-only, com `arquivo:linha`. |
| **Ataque** | Codex | Revisão adversarial de um diff ou de uma decisão. Segurança, corrida, bug de correção. Motor diferente = achados diferentes. |
| **Arquitetura, decisão, teste, deploy, verificação** | Eu | Nunca delego. E sou eu que confirmo ou descarto cada achado dos outros dois. |

**Não delegue:** decisão de arquitetura, deploy, escrever o teste que prova o achado, ou qualquer coisa cujo resultado eu não consiga verificar sozinho.

## Invocação (flags conferidas em 2026-08-31)

**Grok** (`grok` 1.0.13 — modelos: `grok-4.6` default, `grok-4.5`):
```bash
grok -m grok-4.6 --reasoning-effort high --cwd <caminho-do-repo> \
     -p "<prompt>" 2>&1 | tail -100
```
- `-p/--single` é o headless (imprime no stdout e sai). **Sem `-p` ele abre o TUI e pendura em pipe** ("Device not configured").
- Extras: `--output-format json`, `--json-schema '<schema>'` (saída estruturada), `--always-approve`, `--max-turns N`, `--agents <JSON>`.
- `-w/--worktree` **não** combina com `-p`.

**Codex** (`codex` 0.151.0):
```bash
codex exec -m gpt-5.6-sol -c model_reasoning_effort="high" \
     -C <caminho-do-repo> -s read-only \
     -o /tmp/resposta.md "<prompt>"
```
- `codex exec review` = review do repositório inteiro (pronto de fábrica).
- `-s read-only` **força** investigação sem escrita — melhor que pedir "não edite" no prompt.
- `-o/--output-last-message <FILE>` captura a resposta limpa; melhor que `| tail -40`.
- `--skip-git-repo-check` quando o cwd não é repo git. `--output-schema <FILE>` para JSON estruturado.

**Armadilhas de flag (os dois CLIs usam as mesmas letras para coisas diferentes):**

| Flag | No Grok | No Codex |
|---|---|---|
| `-p` | `--single` = o prompt headless | `--profile` = perfil de config ⚠️ |
| `-c` | `--continue` = retomar sessão ⚠️ | `--config key=value` |
| effort | `--reasoning-effort high` | `-c model_reasoning_effort="high"` |
| cwd | `--cwd <dir>` | `-C/--cd <dir>` |

## O loop

1. **Levantar** — Grok varre e devolve os fatos com `arquivo:linha`.
2. **Decidir e implementar** — eu, com teste.
3. **Atacar** — Codex recebe o diff e tenta furar.
4. **Verificar cada achado** — escrevo o teste que reproduz. Não reproduziu, morreu.
5. **Endurecer** — aplico só os achados que sobreviveram.
6. **Segunda rodada** — devolvo ao Codex o diff endurecido **contando o que decidi e por quê**, e mando atacar as decisões. É aqui que aparece o que a primeira rodada não pegou.
7. **Assinar** — CI verde, deploy, e a responsabilidade é minha.

**Paralelize o passo 3 com trabalho meu:** enquanto o Codex ataca o item A, eu implemento o item B. Dispare em background e siga.

## Contrato de prompt — Grok (levantamento)

Grok solto devolve recomendação genérica. Preso ao contrato, devolve dado. Todo prompt de levantamento tem:

1. **Papel + tarefa + read-only explícito** — "Você é engenheiro de segurança web. Tarefa de LEVANTAMENTO (read-only, não edite nada)."
2. **Contexto do stack em uma linha** — o que é o app, onde roda, se é público.
3. **"Preciso de dados concretos extraídos do código, não recomendações genéricas."**
4. **BLOCOS numerados**, um por pergunta, cada um apontando os arquivos onde procurar. Blocos são o que impede a resposta de virar ensaio.
5. **Bloco final = o entregável** — normalmente uma tabela markdown com as colunas já definidas por mim.
6. **Anti-alucinação:** "Cite arquivo:linha sempre. Se não encontrar algo, diga 'não encontrado' em vez de supor."

## Contrato de prompt — Codex (ataque)

1. **Papel adversarial** — "Você é um revisor de segurança adversarial, sem pena."
2. **Contexto do runtime + a contra-advertência.** Não basta descrever o runtime (*"objeto stateful com uma instância por chave, execução serializada, RPC sobre WebSocket"*). É preciso **fechar a saída errada**: *"o runtime já serializa as chamadas a esse objeto; só aponte corrida se ela sobreviver a isso, e diga exatamente como."* Sem essa linha o revisor reinventa a corrida que o runtime já resolve — em teste, a versão sem a contra-advertência chegou a **instruir** o modelo a procurar exatamente o falso positivo que já tinha sido refutado.
3. **Aponte o diff, não cole.** Com `-C <repo>` o Codex lê o arquivo sozinho: *"o diff está em ./diff.txt; leia esse arquivo E as funções inteiras que ele toca, não só as linhas do diff."* Evita inferno de quoting e truncamento. Cole com `$(cat ...)` só quando o arquivo está fora do workspace dele.
4. **Vetores numerados que eu suspeito** — contorno do bloqueio, DoS direcionado, oráculo de existência, corrida, custo de storage. Numerar dirige o ataque em vez de espalhar.
5. **Licença para não achar nada** — "Se não houver furo, diga isso claramente **em vez de inventar**." Sem essa linha ele fabrica achado para justificar a chamada.
6. **Teto de palavras** — "máximo 400 palavras". Corta o relatório inflado.
7. **Na segunda rodada, entregue as decisões e mande atacá-las:** "A corrida que você apontou NÃO se reproduziu — dois testes de integração fecham a contagem certa. Ataque essa conclusão."

## A regra de ouro

**Achado de delegado é hipótese. Reproduza antes de agir.**

Caso real (endurecimento de um fluxo de login): o Codex devolveu 3 achados, um marcado como corrida **crítica**. Ler o código dava razão a ele — o valor era lido antes do `await`. Escrevi dois testes de concorrência (pipeline numa conexão, N conexões paralelas): **não reproduziu**, o runtime já serializava as chamadas àquele objeto. O "crítico" morreu; os outros dois eram reais — um virou ajuste de parâmetro, o outro foi reclassificado como risco aceito e documentado. Antes disso, uma auditoria delegada já tinha trazido outro "crítico" falso: credencial supostamente exposta no código, quando produção lê de secret.

Vale nos dois sentidos: na rodada seguinte o `gpt-5.6-sol` High achou um caminho de credencial realmente desprotegido que eu tinha deixado passar.

## Red flags — pare

- Colar o achado do delegado no relatório sem ter reproduzido.
- Chamar Grok ou Codex sem `--reasoning-effort high` / `model_reasoning_effort="high"` quando o trabalho é de análise.
- Prompt de ataque sem a licença de "não achei nada" — o achado que vier é decoração.
- Prompt de ataque que descreve o runtime mas não fecha a saída errada — sem a contra-advertência, o revisor devolve o falso positivo do manual.
- Chamar o Codex sem fixar `-m` — o modelo default acha menos que o `gpt-5.6-sol` High, e o que ele não achar você não saberá que existiu.
- Prompt de levantamento sem blocos e sem exigir `arquivo:linha` — volta ensaio genérico.
- Delegar a decisão em vez do levantamento.
- Rodar `grok` sem `-p` dentro de pipe e esperar resposta.
