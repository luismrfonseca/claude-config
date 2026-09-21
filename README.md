# claude-config

Espelho versionado de `~/.claude`, com `.gitignore` deny-by-default: só
`skills/`, o `README.md` e os manifestos de `.claude-plugin/` são tracked. Tudo
o resto (credenciais, transcripts, estado local) fica de fora por omissão.

## Também é um marketplace de plugins

Skills em `~/.claude/skills/` vivem no disco de uma máquina. Sessões do Claude
Code na web correm num container efémero na cloud, que não vê esse diretório —
os skills simplesmente não existem lá.

Isso é pior do que parece quando as preferências globais mandam usá-los: as
instruções falham em silêncio. O agente não diz «não tenho o
`julia-second-brain`», faz outra coisa qualquer ou improvisa. Foi exatamente o
que aconteceu a 2026-09-21, com um `@session` pedido numa sessão web.

Os dois manifestos em `.claude-plugin/` resolvem isso sem mover nada: o repo
declara-se marketplace (`marketplace.json`) e plugin (`plugin.json`), e o
`skills/` que já existe é descoberto automaticamente.

```
.claude-plugin/marketplace.json   # catálogo — aponta para "./" (este repo)
.claude-plugin/plugin.json        # manifesto do plugin lf-skills
skills/<nome>/SKILL.md            # cada skill, auto-descoberto
```

O `plugin.json` não lista skills nenhum: acrescentar um é criar
`skills/<nome>/SKILL.md` e mais nada.

## Instalar

```bash
/plugin marketplace add https://github.com/luismrfonseca/claude-config.git
/plugin install lf-skills@lf-skills
```

Repo privado — a máquina precisa de acesso git. Se o `owner/repo` curto falhar
com `Permission denied (publickey)`, usa o URL HTTPS acima; `gh auth setup-git`
instala o credential helper.

## Skills

`auth-rbac` · `codebase-memory` · `domain-architect` · `erp-integration` ·
`graphify` · `headroom` · `michael` · `orchestrator` · `ponytail` ·
`product-owner` · `qa-fitness` · `roberto-code-reviewer`

### Avisos

**`roberto-code-reviewer` também existe como plugin autónomo publicado na
conta.** Desinstala-o antes de instalar o `lf-skills`, senão ficas com duas
cópias do mesmo nome vindas de plugins diferentes, e a daqui deixa de receber
as atualizações que fizeres na outra sem nada avisar.

**`julia-second-brain` não está aqui.** As preferências globais referem-no, mas
não existe neste repo — só na máquina local. Copia
`~/.claude/skills/julia-second-brain/` para `skills/` para o trazer.

**`ponytail-review`, `ponytail-audit` e `ponytail-debt` não são skills
separados** — `skills/ponytail/` só tem `SKILL.md`. São modos dentro do
`ponytail`.
