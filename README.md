# orquestrando-grok-e-codex

Skill do Claude Code para delegar trabalho a um **segundo motor de IA** rodando na mesma máquina — os CLIs `grok` e `codex` — sem abrir mão de quem assina o resultado.

## A dor

Você instala o segundo CLI, manda ele revisar seu código, e recebe de volta um relatório confiante com um achado "crítico". Aí ou você acredita e refatora à toa, ou desconfia e joga tudo fora — e nos dois casos a delegação virou teatro.

O ganho de chamar outro motor não é velocidade. É **viés diferente**: um modelo treinado de outro jeito enxerga o que o seu não enxerga. O custo é que ele também inventa com a mesma confiança. É por isso que a skill inteira gira em torno de uma regra:

> **Eles produzem, eu verifico e assino.** Achado de delegado é hipótese, não fato, até ser reproduzido no código.

## O que tem dentro

| Seção | Conteúdo |
|---|---|
| Divisão de papéis | Grok = levantamento factual em volume · Codex = ataque adversarial · Claude = arquitetura, teste, deploy e verificação |
| Invocação | Linhas de comando headless prontas, com as flags conferidas contra os CLIs |
| Armadilhas de flag | `-p` é prompt no Grok e `--profile` no Codex; `-c` é `--continue` no Grok e `--config` no Codex |
| O loop | 7 passos: levantar → implementar → atacar → **verificar** → endurecer → segunda rodada → assinar |
| Contratos de prompt | O padrão de BLOCOS numerados do Grok e o de vetores de ataque do Codex, com as linhas anti-alucinação que fazem a diferença |
| Red flags | Os sinais de que a delegação está virando teatro |

As duas linhas que mais mudaram resultado, se você só quiser levar isso: **dar licença explícita para não achar nada** ("se não houver furo, diga isso em vez de inventar") e **fechar a saída errada** ao descrever o runtime, para o revisor não devolver o falso positivo do manual.

## Instalação

Nível usuário (vale em todos os projetos):

```bash
git clone https://github.com/mgbn01/orquestrando-grok-e-codex-skill.git \
  ~/.claude/skills/orquestrando-grok-e-codex
```

Ou nível projeto, trocando o destino por `.claude/skills/orquestrando-grok-e-codex` dentro do repo.

Não precisa invocar na mão: o Claude Code descobre a skill pela descrição e a carrega quando o contexto bate. Em teste, um agente que só recebeu o cenário — sem nenhuma menção à skill — a invocou sozinho.

## Pré-requisitos

- [`grok`](https://grok.com) CLI autenticado — validado na 1.0.13
- [`codex`](https://github.com/openai/codex) CLI autenticado — validado na 0.151.0

Nenhum dos dois é obrigatório: só o Codex já dá o loop de ataque, só o Grok já dá o de levantamento.

Ambos os CLIs mudam flag entre versões. Se algum comando falhar, confira o `--help` antes de assumir que a skill está errada.

## Procedência

A metodologia não foi inventada: foi extraída de uma sessão real de hardening de segurança, onde o loop pegou dois achados legítimos, matou um "crítico" falso que não se reproduziu em teste, e revelou um caminho de credencial desprotegido que tinha passado batido.

Depois disso ela foi testada contra um controle — um agente resolvendo o mesmo cenário **sem** a skill. O controle foi forte, e boa parte da skill não estava ensinando nada: estava documentando o que o modelo já faz sozinho. O que sobrou de delta real está em [`docs/teste-basico.md`](docs/teste-basico.md), junto com os limites desse teste.

## Idioma

Escrita em português. Os contratos de prompt funcionam igual em qualquer idioma — o que importa é a estrutura, não a língua.

## Licença

MIT. Veja [LICENSE](LICENSE).

Projeto independente, sem afiliação com xAI, OpenAI ou Anthropic.
