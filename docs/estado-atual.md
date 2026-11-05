# Estado nesta máquina

Zenbook 14 UX3405MA, Fedora 45. Este arquivo descreve o que está instalado e a decisão de uso. O patch em si continua explicado no [README](../README.md).

Endereço Bluetooth do fone fica de fora de propósito. Os nomes abaixo usam o padrão do PipeWire; o endereço real aparece em `pactl list cards`.

## Versões

| Peça | Versão |
| --- | --- |
| Kernel | 7.2.15-300.fc45 |
| PipeWire | 1.6.11 |
| WirePlumber | 0.5.19 |
| BlueZ | 5.87 |
| Firmware SOF | IPC4 2.15.0.1 |
| Topologia SOF | `sof-hda-generic-2ch.tplg` |

O áudio PCI é o Meteor Lake-P HD Audio Controller (`8086:7e28`), codec Realtek ALC294. O kernel sobe `sof-audio-pci-intel-mtl` com o machine driver `skl_hda_dsp_generic`. A tabela NHLT declara 2 microfones digitais, por isso a topologia genérica de 2 canais.

## Uso escolhido

Alto-falante: Baseus Bass EP10 Pro em A2DP (AAC). Microfone: o digital do notebook, não o do fone.

O WirePlumber guarda isso em `~/.local/state/wireplumber/` e restaura no próximo login:

- `sm-settings`: `bluetooth.autoswitch-to-headset-profile=false`
- `default-nodes`: saída `bluez_output.<MAC_>.1`, entrada `alsa_input.pci-0000_00_1f.3-platform-skl_hda_dsp_generic.HiFi__Mic1__source`
- `default-profile`: o cartão do Baseus em `a2dp-sink`

Com a troca automática desligada, abrir um microfone não empurra o fone para HFP. O source falso `bluez_input.<MAC>` some enquanto o perfil é A2DP. Para conferir:

```bash
wpctl settings bluetooth.autoswitch-to-headset-profile
pactl get-default-sink
pactl get-default-source
pactl list cards | awk '/bluez_card/,/Active Profile/'
```

O valor salvo tem de ser `false` e o perfil ativo `a2dp-sink`. Isso é estado do usuário. Apagar `~/.local/state/wireplumber/` ou resetar o WirePlumber desfaz.

Para voltar a usar o microfone do fone:

```bash
wpctl settings -s bluetooth.autoswitch-to-headset-profile true
```

## Plugin local do 1.6.11

A atualização 1.6.9 para 1.6.11 manteve os drop-ins apontando para um `libspa-bluez5.so` compilado contra 1.6.9, com `SPADATADIR` num prefixo `/tmp` que já não existia. O sintoma no journal era `failed to load bluez-hardware.conf` e, na captura, `Failure in Bluetooth audio transport`.

O plugin foi recompilado a partir da tag 1.6.11 com `patches/pipewire-1.6.11-hfp-pending-sco.patch`. Os pacotes do sistema não foram substituídos. Três drop-ins iguais carregam o plugin e devolvem os dados SPA para o caminho da distribuição:

`~/.config/systemd/user/{pipewire,wireplumber,pipewire-pulse}.service.d/99-hfp-sco-fix.conf`

```ini
[Service]
Environment=SPA_PLUGIN_DIR=%h/.local/share/pipewire-hfp-sco-fix/spa
Environment=SPA_DATA_DIR=/usr/share/spa-0.2
```

O diretório `spa/bluez5/` mistura o `libspa-bluez5.so` local e codecs reconstruídos (mSBC, CVSD, SBC) com links para os codecs do sistema. Um `dnf upgrade` do PipeWire pode tornar esse `.so` incompatível de novo.

Confirmação de que o processo carregou o arquivo local:

```bash
pid=$(systemctl --user show -p MainPID --value wireplumber)
grep pipewire-hfp-sco-fix/spa/bluez5/libspa-bluez5.so /proc/$pid/maps
```

O binário local contém a string `reusing pending transport`.

## O que o teste do Teams mostrou

Uma captura direta com `pw-record` no 1.6.11 trocou A2DP para `headset-head-unit`, gravou áudio e voltou para A2DP, sem `Too many links` e sem `Failure in Bluetooth audio transport`.

No Teams o caminho foi outro. O Chrome já tocava no fone em A2DP quando o microfone do Baseus foi selecionado. Às 10:32:50 de 5 de novembro de 2026 o journal registrou `Failure in Bluetooth audio transport` no transporte HFP e os nós foram para erro. O medidor do Teams ficou parado. Uma amostra feita em seguida, fora do Chrome, ainda tinha sinal no source mSBC. O fluxo `RecordStream` do Chrome depois desapareceu, e os dois medidores do Teams pararam porque o aplicativo soltou a captura.

O patch só reaproveita o SCO quando o codec é o mesmo e a conexão ainda está pendente (`fd >= 0`, `requesting`, `err == -EINPROGRESS`). A troca do Teams sai de A2DP para mSBC com reprodução já aberta, então não entra nessa condição. O relato foi feito no [issue 5506](https://gitlab.freedesktop.org/pipewire/pipewire/-/work_items/5506), que já descrevia o `AT+BCC` no meio de um eSCO.

Outro efeito separado: se o PipeWire reinicia com o Teams aberto, o processo `audio.mojom.AudioService` continua com o memfd antigo do Pulse. A tela de entrar mostra microfone e alto-falante em "Nenhum". Fechar o aplicativo de verdade e abrir de novo recria o processo.

## Microfone digital

O dispositivo "Meteor Lake-P HD Audio Controller Digital Microphone" é o PCM `hw:sofhdadsp,6`, estéreo, s32, 48 kHz. O driver é o SOF previsto para essa plataforma, não um fallback.

No DSP, o DRC e o beamformer TDFB estão ligados, com o feixe em 0°. Isso é o processamento da topologia genérica de 2 microfones.

Não há equivalência com o Windows. Lá entram Intel Smart Sound APO, Windows Studio Effects / Voice Focus e o ajuste da ASUS. Aqui não há cancelamento de eco, nem supressão de ruído por NPU, nem filtro do PipeWire nesse source. O controle `Mic Boost` do mixer é do ALC294, o microfone analógico (`Mic2`), não deste digital.
