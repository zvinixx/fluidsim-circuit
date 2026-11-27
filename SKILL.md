---
name: fluidsim-circuit
description: "Analyse, validate and safely edit FluidSIM 4 (Festo) .ct circuit files: decode the container, inspect components and properties, rename labels losslessly, and verify whether a file still opens in FluidSIM. Use when the request mentions 'FluidSIM', '.ct file', 'hydraulic circuit file', 'inspect a circuit', 'rename a circuit component', 'validate a .ct' or 'why does FluidSIM say unknown file format'."
license: MIT
metadata:
  author: zvinixx
  version: "2.0.0"
  fluid_sim: "4.x Hydraulics"
---

# Skill: FluidSIM `.ct` — análise, edição segura e validação

Ferramentas e conhecimento sobre o formato `.ct` do **FluidSIM 4**, com uma
regra de honestidade acima de tudo: **verificar antes de afirmar**.

**Autor da engenharia reversa: [zvinixx](https://github.com/zvinixx).**

---

## Leia isto primeiro: o que é possível hoje

| Objetivo | Estado real | Evidência |
|---|---|---|
| Inspecionar um `.ct` (componentes, propriedades, textos) | ✅ funciona | parser consome 100% dos bytes |
| Renomear rótulos sem corromper | ✅ funciona | abre no FluidSIM com o desenho intacto |
| Validar se um `.ct` abre | ✅ funciona | oráculo por janela MDI |
| Trocar a base por outra do catálogo | ✅ funciona | 201/201 abrem |
| **Escrever um `.ct` do zero** | ❌ **não funciona** | ver §5 |
| **Inserir um componente novo num circuito** | ❌ **não funciona** | ver §5 |

> Se o pedido for "crie um circuito novo do zero", a resposta correta é:
> **não é possível com o conhecimento atual**, e explicar o motivo (§5).
> Não invente um arquivo que o FluidSIM vai rejeitar.

---

## §1 — Onde ficam os arquivos

```
D:\Hydraulic\
├── ct\        circuitos de exemplo (tp501, tp502, tp601, tp602, Shows, ...)
├── sym\       biblioteca de símbolos
│   ├── hyd\actuator\     cilindros (a1.ct = dupla ação)
│   ├── hyd\valve\        válvulas
│   ├── hyd\supply\       bomba, tanque, filtro, acumulador
│   ├── hyd\sensor\       manômetro, pressostato
│   ├── electr\switch\    chaves (Manually, Proximit, General)
│   ├── electr\relay\     relés
│   └── grafcet\          step, transition, action
├── bin\fl_sim_h.exe      o aplicativo
└── misc\                 configurações
```

Cada pasta tem um `pos.txt` descrevendo a grade da biblioteca.

---

## §2 — Inspecionar um arquivo

Decodifique e leia as propriedades. Implementação completa em §6.

```python
import ctcodec
info = ctcodec.describe('circuito.ct')
print(info['tokens'], info['coberto'])
for s in ctcodec.strings('circuito.ct', 5):
    print(s)
```

O que sai: `BEGIN_FS`, `ENDCT`, `BEGINDSD`, `GRAPHRECT`, `description_FEST`,
`Hyd.Actuator`, `DAMPING_ADJ_L`, `x_pos`, `stroke`, `gate1`, etc.

**Nomes de propriedade que importam** (extraídos das *format strings* do
executável):

| Propriedade | Significado |
|---|---|
| `x_pos` / `y_pos` | posição do símbolo na folha |
| `stroke` | curso do cilindro (mm) |
| `length` | comprimento desenhado |
| `MASS` | massa da carga (kg) |
| `DAMPING_ADJ_L` | amortecimento |
| `only_top_rule` / `both_sides_rule` | regras de desenho |
| `f_offset_x` / `f_offset_y` | deslocamento de rótulo |
| `pos_l_<n>` / `pos_r_<n>` | posição de sensor de fim de curso |
| `sig_color` | cor do sinal |
| `PISTON_COUNT`, `PISTON_R`, `HUB` | geometria do cilindro |

---

## §3 — Renomear rótulos (operação segura)

Regra de ouro: **o texto novo precisa ter exatamente o mesmo número de bytes**
que o antigo. Complete com espaços.

```python
import ctcodec
n = ctcodec.patch_text(
    'entrada.ct', 'saida.ct',
    [('Zylinder, doppeltwirkend', 'Cilindro principal     ')])
```

Verificado: o arquivo abre e o componente continua desenhado corretamente.
Com tamanho diferente o arquivo ainda abre, mas o desenho pode corromper.

---

## §4 — Validar de verdade

Validação por estrutura **não basta** (um arquivo truncado passa na checagem
de tokens e abre vazio). A validação confiável é abrir no FluidSIM e olhar a
janela:

| Sinal | Significado |
|---|---|
| janela MDI filha com o nome do arquivo | abriu |
| nenhuma janela filha | abriu vazio — o conteúdo não foi entendido |
| diálogo "Unerwartetes Dateiformat" / "Unknown file format" | rejeitado |

No Windows, o teste automatizado está em §7 (`oracle.ps1`).

---

## §5 — Por que escrever do zero não funciona

### O que sabemos com certeza

* Todo o arquivo passa por `XOR 0x9A` (função em `0x41AEC0`, chave em
  `0x5c544c`).
* Depois disso é um fluxo LZ77 **sem cabeçalho**: o stream começa no byte 0.
  Prova: zerar os primeiros 3 bytes já produz "formato inesperado"; manter
  intactos os primeiros 3 e bagunçar o resto também.
* Gramática dos tokens (confirmada: o parser consome 100% dos bytes em todos
  os 201 arquivos testados):

  ```
  byte >= 0x80  ->  literal de (b - 0x80) bytes
  byte <  0x80  ->  código de 16 bits, BIG-ENDIAN
        comprimento = (code >> SHIFT) + 1
        offset      = code & (W - 1)
        comprimento == 1  ->  RLE (valor no byte seguinte, repetições = offset)
        comprimento == 2  ->  comprimento ESTENDIDO (byte seguinte = tamanho)
        comprimento >= 3  ->  cópia do ring buffer
  ```

  Modo B (padrão): `W = 2048`, `SHIFT = 11`. Modo A: `W = 512`, `SHIFT = 9`.
  O buffer é alocado com `0x7FF8` bytes; o índice de escrita avança e sofre
  *flush* quando `lp + len >= 0x7FF8`.

### O que ainda não sabemos

O decodificador reconstrói **quase tudo** (cabeçalhos de seção, propriedades,
descrições, coordenadas) mas **não byte a byte**: sobram trechos onde o ring
reciclado produz lixo. Consequência prática:

| Tentativa | Resultado |
|---|---|
| copiar um `.ct` real | abre |
| renomear rótulo (mesmo tamanho) | abre |
| decodificar + reencodar só com literais | abre **vazio** |
| montar o texto do zero | abre **vazio** ou é rejeitado |

Ou seja: o FluidSIM **aceita** um arquivo só com literais (a camada de
container está correta), mas o **conteúdo** precisa estar íntegro — e o nosso
decodificador não recupera o conteúdo com fidelidade total.

### O caminho para fechar isso

1. **Depuração dinâmica** — executar `fl_sim_h.exe` sob depurador, colocar
   breakpoint em `0x41B8FD` (logo após a descompressão) e despejar o buffer
   `[0x5d1db0]`. Isso dá o plaintext exato, sem dedução.
2. **Reimplementar o compressor** (`0x41A600` / `0x41A4E0`) e validar
   round-trip byte a byte com esse plaintext.
3. Só então será possível desenhar circuitos novos.

---

## §6 — Codec de referência

Dependência zero. Copie como `ctcodec.py`.

```python
#!/usr/bin/env python3
"""Codec dos arquivos .ct do FluidSIM 4."""
KEY = 0x9A


def xor(data, key=KEY):
    return bytes(b ^ key for b in data)


def read(path):
    with open(path, 'rb') as f:
        return f.read()


def write(path, data):
    with open(path, 'wb') as f:
        f.write(data)


def stream_info(stream, shift=11, ring=2048):
    """Lista de tokens com posicao e tamanho."""
    mask = ring - 1
    toks = []
    i = 0
    n = len(stream)
    while i < n:
        b = stream[i]
        if b & 0x80:
            cnt = b - 0x80
            if i + 1 + cnt > n:
                break
            toks.append({'kind': 'lit', 'pos': i, 'size': 1 + cnt,
                         'data': stream[i + 1:i + 1 + cnt]})
            i += 1 + cnt
            continue
        if i + 2 > n:
            break
        code = (stream[i] << 8) | stream[i + 1]
        length = (code >> shift) + 1
        offset = code & mask
        if length == 1:
            if i + 3 > n:
                break
            toks.append({'kind': 'rle', 'pos': i, 'size': 3,
                         'value': stream[i + 2], 'count': offset})
            i += 3
            continue
        if length == 2:
            if i + 3 > n:
                break
            toks.append({'kind': 'copy', 'pos': i, 'size': 3,
                         'offset': offset, 'length': stream[i + 2],
                         'extended': True})
            i += 3
            continue
        toks.append({'kind': 'copy', 'pos': i, 'size': 2,
                     'offset': offset, 'length': length})
        i += 2
    return toks


def rebuild(stream, toks, replacements):
    """Troca texto apenas nos literais; copias/RLE ficam intactas."""
    out = bytearray()
    hits = 0
    for t in toks:
        if t['kind'] == 'lit':
            data = t['data']
            for old, new in replacements:
                if old in data:
                    hits += data.count(old)
                    data = data.replace(old, new)
            out.append(0x80 + len(data))
            out += data
        else:
            out += stream[t['pos']:t['pos'] + t['size']]
    return bytes(out), hits


def patch_text(src, dst, replacements):
    """Substitui textos. `replacements` = [(antigo, novo), ...]."""
    if isinstance(replacements, dict):
        replacements = list(replacements.items())
    pairs = [(o.encode('latin1'), n.encode('latin1')) for o, n in replacements]
    raw = xor(read(src))
    toks = stream_info(raw)
    new_stream, hits = rebuild(raw, toks, pairs)
    if hits:
        write(dst, xor(new_stream))
    return hits


def literals(path):
    return [t['data'] for t in stream_info(xor(read(path)))
            if t['kind'] == 'lit']


def strings(path, minlen=4):
    out = []
    for chunk in literals(path):
        txt = ''.join(chr(c) if 32 <= c < 127 else ' ' for c in chunk)
        out.extend(w for w in txt.split() if len(w) >= minlen)
    return out


def describe(path):
    raw = xor(read(path))
    toks = stream_info(raw)
    counts = {}
    for t in toks:
        counts[t['kind']] = counts.get(t['kind'], 0) + 1
    return {'arquivo': path, 'tamanho': len(raw), 'tokens': counts,
            'coberto': sum(t['size'] for t in toks) == len(raw)}
```

---

## §7 — Oráculo de validação (Windows)

`oracle.ps1` — retorna `ABRIU`, `VAZIO` ou `ERRO`.

```powershell
param([Parameter(Mandatory=$true)][string]$CtPath, [int]$WaitMs = 14000)

$exe = "D:\Hydraulic\bin\fl_sim_h.exe"
$abs = (Resolve-Path $CtPath).Path
$name = [System.IO.Path]::GetFileName($abs)

Add-Type @"
using System;
using System.Text;
using System.Collections.Generic;
using System.Runtime.InteropServices;
public class WDet {
  [DllImport("user32.dll")] static extern bool EnumWindows(Cb cb, IntPtr l);
  [DllImport("user32.dll")] static extern bool EnumChildWindows(IntPtr p, Cb cb, IntPtr l);
  [DllImport("user32.dll")] static extern uint GetWindowThreadProcessId(IntPtr h, out uint pid);
  [DllImport("user32.dll", CharSet=CharSet.Unicode)] static extern int GetWindowTextW(IntPtr h, StringBuilder s, int n);
  [DllImport("user32.dll")] static extern bool IsWindowVisible(IntPtr h);
  public delegate bool Cb(IntPtr h, IntPtr l);
  public static List<string> Docs(uint pid) {
    var acc = new List<string>();
    EnumWindows(delegate(IntPtr h, IntPtr l) {
      uint wp; GetWindowThreadProcessId(h, out wp);
      if (wp != pid) return true;
      EnumChildWindows(h, delegate(IntPtr ch, IntPtr cl) {
        var t = new StringBuilder(1024); GetWindowTextW(ch, t, t.Capacity);
        if (t.Length > 0) acc.Add(t.ToString());
        return true;
      }, IntPtr.Zero);
      return true;
    }, IntPtr.Zero);
    return acc;
  }
}
"@

$p = Start-Process -FilePath $exe -ArgumentList "`"$abs`"" -PassThru
$verdict = "VAZIO"
$deadline = (Get-Date).AddMilliseconds($WaitMs)
while ((Get-Date) -lt $deadline) {
    Start-Sleep -Milliseconds 600
    $p.Refresh()
    if ($p.HasExited) { $verdict = "ERRO"; break }
    $docs = [WDet]::Docs([uint32]$p.Id)
    $joined = ($docs -join ' | ')
    if ($joined -match 'Unerwartetes Dateiformat|Unknown file format') { $verdict = "ERRO"; break }
    if ($docs | Where-Object { $_ -like "*$name*" }) { $verdict = "ABRIU"; break }
}
if (-not $p.HasExited) { Stop-Process -Id $p.Id -Force }
Write-Output $verdict
```

**Importante:** `ABRIU` sozinho não prova conteúdo correto — confirme com
captura de tela. Um documento vazio também "abre".

---

## §8 — Mapa do executável

| Endereço | Papel |
|---|---|
| `0x41AEC0` | aplica/remove XOR |
| `0x41A600` | compressor |
| `0x41A4E0` | busca de match LZ |
| `0x41AF80` | descompressor |
| `0x41B3A8` | laço do descompressor |
| `0x41B5F7` | leitura do código de 16 bits |
| `0x41B740` | ramo `length == 1` (RLE) |
| `0x41B786` | cópia ring → saída |
| `0x41B83D` | flush final |
| `0x4C8450` | wrapper de descompressão |

| Global | Nome |
|---|---|
| `0x5c544c` | chave XOR (0x9A) |
| `0x5c5450` | modo (0 = B/2048, ≠0 = A/512) |
| `0x5c5454` | `W` |
| `0x5c5456` | `MAXLEN` (64 no A, 16 no B) |
| `0x5c5458` | `SHIFT` |
| `0x5d1d94` | buffer de entrada |
| `0x5d1d98` | ring buffer (`0x7FF8` bytes) |
| `0x5d1d9c` | `lp` (posição de escrita) |
| `0x5d1db0` | buffer de saída |

Biblioteca de símbolos: `sym/hyd/actuator/a1.ct` é o cilindro de dupla ação;
`sym/electr/switch/Proximit/px1..px4.ct` são os sensores de proximidade usados
como fim de curso.

---

## §9 — Frases que ativam esta skill

**Análise**
- "Inspecione este `.ct` e liste os componentes."
- "Quais propriedades tem o cilindro deste circuito?"
- "Extraia os textos legíveis deste arquivo."

**Edição segura**
- "Renomeie o cilindro para 'Atuador de prensa' mantendo o arquivo válido."
- "Troque a descrição do componente por 'Cilindro hidráulico'."

**Diagnóstico**
- "Por que o FluidSIM diz 'formato inesperado' neste arquivo?"
- "Valide este `.ct` e me diga se abre com conteúdo."

**Expectativa (responder com honestidade)**
- "Crie um circuito novo do zero" → explicar §5, oferecer editar um existente.
- "Adicione um fim de curso a este circuito" → explicar que inserir
  componente não é suportado; sugerir editar em um que já tenha.

---

## §10 — Checklist

- [ ] o pedido é análise/edição (possível) ou criação do zero (não possível)?
- [ ] se criação: explique o limite antes de gerar qualquer arquivo
- [ ] se edição: mesmo comprimento nos rótulos
- [ ] validar com `oracle.ps1`
- [ ] confirmar visualmente (documento vazio ≠ sucesso)
- [ ] relatar o que foi feito e o que ficou de fora

---

MIT © zvinixx
