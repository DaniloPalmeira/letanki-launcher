# letanki-launcher (repositorio de publicacao)

Arquivos para o repositorio publico `DaniloPalmeira/letanki-launcher`. Ele
hospeda duas coisas que o launcher le sozinho:

- `config.json` (branch `main`): a config de bootstrap ASSINADA. O launcher a
  busca em ordem em
  `raw.githubusercontent.com/DaniloPalmeira/letanki-launcher/main/config.json`,
  `cdn.jsdelivr.net/gh/DaniloPalmeira/letanki-launcher@main/config.json` e
  `res.letanki.com/launcher/config.json` (2,5 s cada). O jsDelivr guarda cache
  por ate 12 h: o raw e o primeiro de proposito.
- Releases com `latest.json` + o setup novo + o `.sig`: o atualizador le
  `releases/latest/download/latest.json` e, se falhar,
  `res.letanki.com/launcher/latest.json` (a lista pode ser trocada pela config).

## O instalador do site NUNCA muda

O objetivo e reputacao no SmartScreen e nos antivirus: o arquivo baixado do
site (`LegacyTankiSetup.exe`) tem sempre os mesmos bytes e a mesma assinatura.
Ele instala a versao que tinha; na PRIMEIRA abertura o launcher ja instala a
mais nova pelo atualizador (download feito pelo proprio launcher, sem a marca
"baixado da internet", logo sem SmartScreen), e dali em diante atualiza sozinho.

O que pode mudar sem instalador novo (tudo pela config assinada, com o valor
embutido de reserva): endereco do jogo e dos recursos, hosts liberados na
janela, servidores, app/textos do Discord, enderecos do atualizador, versao
minima, e a versao do jogo (que vem do site).

So exige regerar `LegacyTankiSetup.exe`:

- a chave PUBLICA do atualizador (`plugins.updater.pubkey`) ou a da config
  (`src-tauri/config-pubkey.txt`) mudar -- perdeu ou vazou a privada;
- o certificado Authenticode do instalador mudar (expirou/trocou);
- a identidade do app mudar (`identifier` `com.letanki.launcher`,
  `productName`/pasta de instalacao) -- nao mude;
- TODAS as tres fontes de config E os dois enderecos do atualizador morrerem
  ao mesmo tempo (instalacoes antigas ficam presas na versao que tem).

Nao mude: nome do arquivo (`LegacyTankiSetup.exe`), `identifier`,
`productName`, `installMode currentUser`.

## Chaves (fora do git, guarde com backup)

```
# atualizador (minisign do Tauri)
npx tauri signer generate -w C:\chaves\letanki-updater.key
#   -> a .pub vai em src-tauri/tauri.conf.json plugins.updater.pubkey
# config (Ed25519)
node publish/sign-config.mjs gen C:\chaves
#   -> a publica vai em src-tauri/config-pubkey.txt
```

As chaves publicas no repositorio sao as REAIS (privadas fora do git). Se trocar
alguma, regere src-tauri/src/testdata/config.signed.json
com a nova, senao o `cargo test` falha).

## Publicar uma versao do launcher

1. Suba `version` em `src-tauri/tauri.conf.json` e `Cargo.toml`.
2. Build com os artefatos do atualizador:
   ```
   $env:TAURI_SIGNING_PRIVATE_KEY_PATH='C:\chaves\letanki-updater.key'
   $env:TAURI_SIGNING_PRIVATE_KEY_PASSWORD='...'
   npm run build:release
   ```
   Sai em `src-tauri/target/release/bundle/nsis/`: `Legacy Tanki_X.Y.Z_x64-setup.exe`
   e o `.sig`.
3. Com Authenticode, deixe o Tauri assinar (`bundle.windows.signCommand` ou
   `certificateThumbprint`): ele assina o setup ANTES de gerar o `.sig`. Assinar
   o setup depois invalida o `.sig`.
4. Monte o `latest.json` (exemplo em `latest.example.json`): `version`, `notes`,
   `pub_date` e `platforms.windows-x86_64` com `url` do setup no release e
   `signature` = CONTEUDO do `.sig`.
5. Crie o release `vX.Y.Z` no GitHub com o setup e o `latest.json` (marque como
   latest) e copie os mesmos dois arquivos para `res.letanki.com/launcher/`.
6. O `LegacyTankiSetup.exe` do site NAO muda. So na primeira publicacao (ou
   num dos casos acima): copie o setup daquela versao com esse nome fixo para o
   site e nunca mais o substitua.

## Mudar a config

1. Edite o payload (exemplo: `config.payload.example.json`) e SUBA `version`
   (o launcher recusa versao menor que a ultima que aceitou).
2. `node sign-config.mjs sign payload.json C:\chaves\config-key.pem config.json`
3. `node sign-config.mjs verify config.json <publica>`
4. Commit do `config.json` na `main` do letanki-launcher e copia em
   `res.letanki.com/launcher/config.json`.

Vale na abertura seguinte (o launcher usa a guardada e busca em segundo plano);
o app do Discord troca na hora. `minLauncher` acima da versao instalada faz o
atualizador instalar ja na abertura.

## Assinar o instalador (Authenticode)

Com um certificado de assinatura de codigo (OV/EV, de uma CA; nao gere um
autoassinado para o publico -- nao da reputacao):

```
signtool sign /fd SHA256 /tr http://timestamp.digicert.com /td SHA256 /f cert.pfx /p SENHA LegacyTankiSetup.exe
```

ou, para o Tauri assinar exe e setup no build, em `tauri.conf.json`:
`bundle.windows.certificateThumbprint` + `timestampUrl` (certificado no
repositorio do Windows) ou `bundle.windows.signCommand`. SEMPRE com carimbo de
tempo (`/tr`): o setup continua valido depois que o certificado expira, e e isso
que deixa o mesmo arquivo servir para sempre.
