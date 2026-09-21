# PipeWire HFP/SCO fixes

Correções experimentais para o backend Bluetooth HFP nativo do PipeWire. Os problemas podem fazer o microfone de fones Bluetooth desaparecer ou gravar áudio vazio em aplicações como WhatsApp Web quando o sistema troca de A2DP para HFP/mSBC ou repete rapidamente o ciclo de captura.

> Status: experimental. Testado com Fedora 44, PipeWire 1.6.9, WirePlumber 0.5.17, BlueZ 5.87 e Baseus Bass EP10 Pro. Ainda não é uma correção oficial do PipeWire.

## O problema

Durante a negociação HFP, o headset responde `AT+BCS=2` para selecionar mSBC. O backend original recria o transporte SCO mesmo quando já existe uma conexão SCO pendente. Em uma janela de corrida, o kernel retorna:

```text
Too many links (-31)
```

O resultado é um transporte HFP sem áudio, que aparece para o navegador como microfone ausente ou silencioso.

Há também uma segunda falha no ciclo de vida: uma liberação normal durante a troca HFP→A2DP era publicada como `ACTIVE→IDLE`, mas o `media-sink` a contabilizava como falha. Depois de três ocorrências próximas, o limite interno de tentativas podia recusar a próxima aquisição SCO. Isso reproduzia o sintoma de o WhatsApp parar de captar depois de algumas mensagens.

## O que o patch faz

O patch altera pontualmente quatro arquivos internos do plugin Bluetooth:

1. Reutiliza o transporte apenas quando o codec é o mesmo e a conexão SCO ainda está realmente pendente.
2. Mantém o comportamento original para transportes ativos, inválidos ou com codec diferente.
3. Marca somente uma liberação solicitada pelo próprio gerenciador; quedas inesperadas continuam sendo tratadas como erro.
4. Não altera o kernel, o driver `btusb`, os codecs nem a negociação normal A2DP.

## Instalação local no Fedora

A instalação abaixo não substitui arquivos do sistema. Ela compila somente o plugin Bluetooth e ativa-o para os serviços do usuário.

### 1. Dependências e fonte

Use uma árvore do PipeWire que corresponda exatamente à versão instalada:

```bash
sudo dnf install dnf-plugins-core git meson ninja-build gcc gcc-c++ pkgconf-pkg-config
sudo dnf builddep pipewire

git clone --branch 1.6.9 --depth 1 \
  https://gitlab.freedesktop.org/pipewire/pipewire.git pipewire-1.6.9
cd pipewire-1.6.9
git apply /caminho/para/pipewire-hfp-sco-fix/patches/pipewire-1.6.9-hfp-pending-sco.patch
```

Se a versão do sistema não for 1.6.9, use a tag correspondente e gere/valide um patch específico para ela. Não aplique este arquivo cegamente em outra versão.

### 2. Compilação

```bash
meson setup build --buildtype=debugoptimized
meson compile -C build spa/plugins/bluez5/libspa-bluez5.so
```

O alvo pode exigir as opções de configuração usadas pela distribuição. O importante é que o arquivo abaixo seja gerado:

```text
build/spa/plugins/bluez5/libspa-bluez5.so
```

### 3. Ativação sem sobrescrever o sistema

Defina os caminhos para a árvore compilada e para os plugins SPA instalados pela distribuição. No Fedora x86_64, o diretório normalmente é `/usr/lib64/spa-0.2`:

```bash
BUILD_DIR="$PWD/build"
SYSTEM_SPA_DIR=/usr/lib64/spa-0.2
STAGE_DIR="${XDG_RUNTIME_DIR:-/tmp}/pipewire-hfp-sco-fix-spa"

mkdir -p "$STAGE_DIR"
for plugin_dir in "$SYSTEM_SPA_DIR"/*; do
  name=$(basename "$plugin_dir")
  [ "$name" = bluez5 ] && continue
  ln -sfn "$plugin_dir" "$STAGE_DIR/$name"
done
ln -sfn "$BUILD_DIR/spa/plugins/bluez5" "$STAGE_DIR/bluez5"

systemctl --user set-environment SPA_PLUGIN_DIR="$STAGE_DIR"
systemctl --user restart pipewire.service wireplumber.service pipewire-pulse.service
```

Confirme que o plugin local foi carregado:

```bash
pid=$(pgrep -n wireplumber)
grep bluez5/libspa-bluez5.so "/proc/$pid/maps"
```

### 4. Rollback

Para voltar aos plugins da distribuição:

```bash
systemctl --user unset-environment SPA_PLUGIN_DIR
systemctl --user restart pipewire.service wireplumber.service pipewire-pulse.service
```

Não é necessário reiniciar o computador. O navegador deve ser totalmente fechado e reaberto depois do reinício do PipeWire.

## Validação

O roteiro detalhado está em [docs/validation.md](docs/validation.md). A validação mínima deve confirmar:

- perfil A2DP muda para `headset-head-unit` ao iniciar uma captura;
- a gravação contém áudio;
- o log mostra `reusing pending transport`, `sco_ready` e `acquire complete`;
- não aparece `Too many links (-31)` nem `SCO socket error`;
- quatro capturas consecutivas continuam funcionando;
- uma liberação normal não gera `Failure in Bluetooth audio transport`.

## Limitações conhecidas

- O patch foi validado em um único headset e ainda precisa de testes com outros fabricantes.
- O cenário mais provável é HFP como Audio Gateway usando mSBC; outros papéis, CVSD e codecs adicionais precisam ser testados.
- O patch é específico da versão 1.6.9 neste repositório.
- O uso de `SPA_PLUGIN_DIR` é local à sessão do usuário e pode precisar ser reativado após logout/reboot.

## English summary

This repository contains experimental PipeWire 1.6.9 fixes for native HFP/SCO. They address both the same-codec mSBC race that could trigger `Too many links (-31)` and the false transport-error accounting during an intentional HFP release. The patch reuses only an explicitly pending same-codec SCO connection and keeps unexpected transport failures on the original error path.

This is not a kernel or headset-driver fix. Please test with your own headset and PipeWire version before reporting success. See the installation and rollback instructions above and the validation checklist in `docs/validation.md`.

## Licença

Material deste repositório: MIT. O patch modifica código do PipeWire; consulte a licença e os avisos da versão correspondente do PipeWire ao redistribuí-lo.
