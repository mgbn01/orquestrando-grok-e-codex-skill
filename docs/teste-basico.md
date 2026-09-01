# Teste básico da skill

Metodologia: mesmo cenário para agentes frescos, um **sem** a skill (controle) e um **com**. Cenário — endurecimento anti força-bruta num fluxo de login sobre um objeto stateful com uma instância por chave e execução serializada, diff salvo em `./diff.txt`, pedir segunda opinião a outro motor antes do deploy. Nenhum agente executou comando; todos entregaram comando + prompt integral + o que fariam com os achados.

## Resultado honesto: o controle foi forte

O agente sem skill acertou sozinho a maior parte: escolheu Codex em vez de Grok, usou `-s read-only` e `model_reasoning_effort="high"`, numerou 9 vetores de ataque, exigiu `arquivo:linha`, mandou marcar "não confirmado" em vez de supor, e disse "não confio cego" antes de aceitar achado.

**Isso significa que boa parte da skill não estava ensinando nada** — estava documentando o que o modelo já faz. O valor real da skill é o delta abaixo.

## O delta que a skill acrescenta

| # | Diferença | Por que importa |
|---|---|---|
| 1 | **Contra-advertência da serialização** | O controle não só omitiu — ele **instruiu** o revisor a caçar a corrida read-then-write, que é exatamente o falso positivo "crítico" já refutado por dois testes de concorrência na sessão real. A skill manda fechar a saída: *"só aponte corrida se ela sobreviver à serialização do runtime, e diga exatamente como."* Maior delta do teste. |
| 2 | **Fixar `-m gpt-5.6-sol`** | O controle não fixou modelo. Na sessão real, o sol 5.6 High achou um caminho de credencial desprotegido que a rodada anterior não pegou. O que o modelo default não achar, você não fica sabendo que existiu. |
| 3 | **Teto de palavras** | O controle pediu 9 vetores + GO/NO-GO sem limite: relatório inflado. A skill impõe "máximo 400 palavras". |
| 4 | **Segunda rodada com as decisões de volta** | O controle re-revisa o diff corrigido, mas não conta ao revisor o que foi decidido e por quê, nem manda atacar essas conclusões. É nessa rodada que aparece o que a primeira não pegou. |
| 5 | **`-o <arquivo>`** | O controle não capturou a saída em arquivo. |

## O que o teste mudou na skill (refactor)

O controle teve **uma ideia melhor que a da skill** e ela foi adotada: em vez de colar o diff com `$(cat ...)`, apontar o caminho do arquivo e deixar o Codex ler sozinho (ele já tem o repo via `-C`). Evita inferno de quoting e truncamento. A colagem inline ficou como fallback para arquivo fora do workspace.

Também foram acrescentados dois red flags: prompt de ataque que descreve o runtime sem fechar a saída errada, e chamada ao Codex sem fixar `-m`.

## Achado lateral: discoverability

O primeiro agente foi lançado **sem** menção à skill, apenas com o cenário — e a invocou por conta própria pela descrição. A descrição dispara no contexto certo sem precisar ser nomeada. (Isso invalidou aquela rodada como controle e obrigou a rodar um controle explicitamente proibido de usar skills — o que está reportado acima.)

## Limites deste teste

Uma repetição por braço, três agentes no total. A `writing-skills` pede 5+ repetições por variante porque amostra única mente. Os itens 1 e 2 da tabela são diferenças de conteúdo factual (o modelo não tem como saber da serialização já refutada nem de qual modelo achou mais), então são robustos. Os itens 3-5 são de forma e mereceriam mais repetições antes de virarem certeza.
