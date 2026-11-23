# Integração AASIAM HUD → site do campeonato

O que o site `aasiam.com.br` precisa ter para o AASIAM HUD enviar as estatísticas de cada mapa direto para a página da série (ex.: `/torneio-cs/serie/R1-M01`).

O HUD é público: qualquer pessoa pode baixá-lo. Por isso o envio só funciona com uma **chave de transmissão**. A organização cria essa chave no admin do site para cada responsável pela transmissão. Ela só dá acesso a estas rotas e não abre o resto do admin (loja, inscrições, homologação).

O HUD já está pronto para este contrato. Enquanto o site não tiver as rotas, o painel do HUD mostra "O site ainda não tem a integração com o HUD".

## 1. Chaves de transmissão (admin do site)

Nova seção no admin, por exemplo "CS2 › Chaves de transmissão":

- **Criar chave.** Pede o nome do responsável (ex.: "Transmissão — Fulano") e, se quiser, uma data de validade.
  - Gera um valor aleatório de pelo menos 32 caracteres, só com `A-Z a-z 0-9 _ -`. Sugestão: `tx_` + 40 caracteres de `crypto.randomBytes(30).toString('base64url')`.
  - A chave aparece **uma vez só**, para copiar e mandar ao responsável.
- **Guardar só o hash** (SHA-256) da chave, mais o nome, a data de criação, o último uso e a validade.
- **Revogar.** Apaga ou desativa a chave na hora. O HUD recebe 401 e pede uma nova.
- **Lista.** Nome, final da chave (últimos 4 caracteres), criada em, último uso e status.

## 2. Autenticação

Todas as rotas abaixo:

- recebem `Authorization: Bearer <chave>`. A chave nunca vai na URL;
- respondem `401 {"error": "..."}` quando a chave é inválida, revogada ou vencida;
- aceitam só JSON e corpo de até 32 KB;
- têm limite de uso, por exemplo 30 requisições por minuto por chave;
- registram cada envio no CS Log com o nome do responsável. Esse nome é o mesmo que hoje vai no cabeçalho `X-CS-Operador`.

O HUD chama o site a partir do servidor local (Node), não do navegador, então não é preciso CORS. O HUD não segue redirecionamentos, para a chave não ir parar em outro endereço. As rotas devem responder direto em `https://www.aasiam.com.br`.

Base: `https://www.aasiam.com.br/api/transmissao`

## 3. Rotas

### `GET /eu`

Confere a chave quando o responsável conecta no HUD.

```json
200 { "ok": true, "operador": "Transmissão — Fulano" }
```

### `GET /series`

Lista as séries que ainda **não estão homologadas** (agendadas, ao vivo ou encerradas aguardando homologação).

```json
200 {
  "series": [
    {
      "id": "R1-M01",
      "titulo": "Rodada 1, mesa 1",
      "status": "ao-vivo",
      "equipes": {
        "A": { "nome": "Elas quer Gin", "tag": "TOMAG",
               "jogadores": [ { "indice": 0, "nome": "Luiz Hoppe", "nick": "Luiz o bigode" } ] },
        "B": { "nome": "Cacha-C4", "tag": "C4", "jogadores": [ ... ] }
      },
      "mapas": [
        { "numero": 1, "nome": "Mirage", "placar": { "A": 13, "B": 1 }, "estatisticas": true },
        { "numero": 2, "nome": "Cache",  "placar": null, "estatisticas": false }
      ]
    }
  ]
}
```

- `jogadores` é a escalação da série, titulares e reservas. `indice` é o mesmo índice que o importador de CSV do admin já usa em `vinculos`.
- `nick` é o nome no jogo informado na inscrição. É o que permite vincular sozinho.
- `estatisticas: true` indica que o mapa já tem estatísticas salvas. O HUD avisa, e um novo envio substitui as anteriores.

### `POST /series/{id}/mapas/{n}/previa`

Mesma regra do "Importar CSV da xplay.gg" do admin (`/series/{id}/importar-csv/previa`), sem gravar nada.

Corpo:

```json
{ "csv": "mapa,partida,equipe,...\n..." }
```

O CSV vem no mesmo formato que o admin já importa: as colunas `mapa, partida, equipe, placar_equipe, placar_adversario, resultado, usuario_id, nickname, steam_id, rounds_played, kills, …, rating_hltv_1_0`, com BOM UTF-8 no início.

```json
200 {
  "ok": true,
  "previa": {
    "mapaNome": "Mirage",
    "rounds": 14,
    "equipes": [
      { "lado": "A", "nome": "Elas quer Gin", "csvEquipe": "Elas quer Gin", "placar": "13–1",
        "jogadores": [ { "csvNick": "Luiz o bigode", "jogador": 0 }, { "csvNick": "novato", "jogador": null } ] }
    ],
    "avisos": []
  }
}
422 { "error": "O CSV é do mapa Inferno, mas o mapa 1 desta série é Mirage." }
```

- `jogador` traz o índice na escalação quando o site consegue vincular sozinho, pelo nick ou pelo SteamID, e `null` quando não consegue. O HUD mostra uma lista para o responsável escolher.
- Use 422 quando o CSV não serve para aquele mapa: mapa diferente, equipes que não são as da série ou placar diferente do já salvo. A mensagem em `error` aparece no painel do HUD.

### `POST /series/{id}/mapas/{n}/estatisticas`

Grava. Faz o mesmo que o `importar-csv` do admin.

Corpo:

```json
{
  "csv": "...",
  "vinculos": { "A": { "novato": 5 }, "B": {} },
  "confirmarAvisos": true
}
```

`vinculos` relaciona o `csvNick` com o índice na escalação. Só vem para quem o responsável vinculou à mão.

Respostas:

| Código | Corpo | Quando |
|---|---|---|
| 200 | `{ "ok": true, "mensagem": "Mapa 1 importado. O site atualiza em até 1 minuto." }` | Gravou. |
| 409 | `{ "avisos": ["..."] }` | Contas que não batem. O HUD mostra os avisos e reenvia com `confirmarAvisos: true` se o responsável confirmar. |
| 409 | `{ "pendentes": { "A": ["novato"] } }` | Algum jogador ficou sem vínculo e o site exige vínculo. |
| 422 | `{ "error": "..." }` | CSV não serve para esse mapa, ou a série já foi homologada. |

## 4. Limites do que a chave pode fazer

- Só lê séries e grava estatísticas de mapas de séries **não homologadas**.
- Não homologa, não reabre, não lança W.O. e não mexe em escalação, inscrição nem em nada fora do campeonato de CS2.
- O resultado oficial continua dependendo da homologação feita no admin, como hoje.

## 5. Como testar

1. Crie uma chave no admin.
2. No HUD, abra **Painel › Estatísticas › Enviar ao site do campeonato**, cole a chave e clique em **Conectar**. Deve aparecer "Conectado como <nome>".
3. Escolha a série e o mapa e clique em **Enviar CSV do mapa ao vivo**. Para uma demo, leia a demo no painel e clique em **Enviar ao site**.
4. Confira a prévia e confirme. A página da série deve mostrar as estatísticas em até 1 minuto.
5. Revogue a chave no admin. O próximo envio do HUD deve dizer "Chave de transmissão inválida ou revogada".
