# Projeto: vídeos dinâmicos com cortes (Reels/Shorts a partir de vídeos longos)

Responda sempre em português do Brasil.

## Objetivo
Pegar vídeos brutos (entrevistas, palestras, falas longas) e transformá-los em vídeos dinâmicos para Instagram/YouTube:
- remover silêncios e pausas (ritmo rápido, estilo Reels/TikTok, sem picotar a fala);
- remover hesitações ("hã", "hum", "é…" arrastado), gaguejos e repetições, usando transcrição com tempo por palavra;
- reenquadrar em vertical 9:16 seguindo o rosto de quem fala;
- zoom de corte (punch-in) alternado a cada corte, para esconder os jump cuts;
- legendas animadas palavra por palavra (estilo CapCut);
- título-gancho animado, nome/cargo de quem fala e crédito da fonte;
- gerar Shorts/Reels com os melhores trechos e, depois, uma versão longa enxuta.

## Vídeo deste projeto  ← PREENCHER
- Arquivo: `brutos/<NOME_DO_ARQUIVO>`
- Origem/link: <URL>
- Licença e crédito obrigatório: <ex.: CC BY 3.0 — "Nome do autor">
- Quem fala (para o letreiro de nome): <NOME> · <cargo>
- Em planos abertos, a pessoa a seguir fica à: <direita | esquerda | centro>
- Nomes próprios/termos que o Whisper pode errar (para o prompt e para corrigir na legenda): <lista>

## Estrutura de pastas
- `brutos/` → vídeos originais (NUNCA editar; trabalhar sempre em cópias/saídas)
- `transcricoes/` → `<nome>.json` (palavras com tempo) e `<nome>.txt` (texto legível com [mm:ss])
- `shorts/` → Reels/Shorts verticais finais (`shorts/_trabalho/` guarda legenda.ass, filtro.txt e log do último render)
- `editados/` → versões longas editadas
- `scripts/` → `transcrever.py` e `gerar_reel.py`
- `fontes/` → TTFs usadas na legenda (copiar `ariblk.ttf`, `arialbd.ttf`, `arial.ttf` de `C:\Windows\Fonts`)
- `modelos/` → `face_detection_yunet_2023mar.onnx`
- `.venv/` → ambiente Python isolado

## Próximos passos
1. Conferir o ambiente (seção "Ambiente") e baixar o vídeo para `brutos/`. Conferir tamanho/duração com `ffprobe`.
2. Transcrever o vídeo inteiro com `scripts/transcrever.py`.
3. Ler a transcrição e propor 3–5 trechos com gancho (ver "Como escolher trechos"); fazer **um Reel de teste** primeiro.
4. Mostrar ao usuário e ajustar o estilo (rigor do corte, cor/fonte/tamanho da legenda, intensidade do zoom, título).
5. Com o estilo aprovado, gerar os demais Reels e a versão longa enxuta em `editados/`.

Se `scripts/` já existir com os dois scripts, use-os. Se não existir, crie-os seguindo a especificação desta página.

Os scripts vieram de um projeto anterior. Antes de usar, troque o que é específico do outro vídeo:
- `transcrever.py`: o texto de `initial_prompt`, que tem os nomes do vídeo anterior.
- `gerar_reel.py`:
  - `VIDEO_PADRAO` e `TRANSC_PADRAO`, ou passe `--video`/`--transcricao`;
  - os padrões de `--nome`, `--cargo` e `--credito`;
  - o dicionário `CORRECOES`;
  - `ALVO_ABERTO`.

---

## Ambiente (testado no Windows 11; adaptar se for outro SO)
- **FFmpeg 7.x "full build"** (ex.: gyan.dev) no PATH; precisa ter `libass` (filtro `subtitles`) e `loudnorm`. Confira com `ffmpeg -filters | grep -E "subtitles|loudnorm"`.
- **Python 3.10+** com ambiente isolado:
  `python -m venv --system-site-packages .venv` e `.venv/Scripts/python -m pip install faster-whisper opencv-python numpy`
  (`--system-site-packages` serve para aproveitar um PyTorch com CUDA que já esteja instalado.)
- **GPU NVIDIA (opcional, recomendado):** o faster-whisper (ctranslate2) precisa de cuBLAS 12 e cuDNN 9.
  - Se houver PyTorch com CUDA 12 instalado: no script, `os.add_dll_directory(os.path.join(os.path.dirname(torch.__file__), "lib"))` antes de importar o faster_whisper.
  - Senão: `pip install nvidia-cublas-cu12 nvidia-cudnn-cu12` e adicionar as pastas `bin` desses pacotes ao PATH/DLL directory.
  - Sem GPU: `device="cpu", compute_type="int8"` (bem mais lento; considere o modelo `medium` ou `small`).
- **Detector de rosto YuNet** (OpenCV ≥ 4.8, `cv2.FaceDetectorYN`):
  `curl -L -o modelos/face_detection_yunet_2023mar.onnx https://github.com/opencv/opencv_zoo/raw/main/models/face_detection_yunet/face_detection_yunet_2023mar.onnx`

## Armadilhas já conhecidas (não repetir)
- **Wikimedia Commons responde HTTP 429** para downloads por script (pede User-Agent com contato). Não insistir com retry em loop (só prolonga o bloqueio) e não colocar e-mail do usuário no User-Agent sem permissão: peça para o usuário baixar pelo navegador e salvar em `brutos/`.
- **Ao rodar curl em background, não termine o comando com `ls`/`echo`**: isso mascara o código de saída. Use `curl --fail ...; rc=$?; echo "exit=$rc"; exit $rc`.
- **`drawtext` do FFmpeg pode dar segfault** quando o fontconfig não está configurado (Windows). Não usar drawtext; todo texto vai por legenda ASS (libass) com `fontsdir`.
- **Caminhos no filtro `subtitles`**: o `D:` precisa de escape. Mais simples: rodar o ffmpeg com `cwd` na pasta de trabalho e usar caminhos relativos (`subtitles=legenda.ass:fontsdir=fontes`), copiando as fontes para `fontes/` dentro dessa pasta.
- **PyAV incompatível com faster-whisper 1.2.x** (`open() got an unexpected keyword argument 'metadata_errors'`). Não deixar o faster-whisper ler o vídeo: decodificar o áudio com FFmpeg (`-ac 1 -ar 16000 -f f32le -`) e passar o array numpy para `model.transcribe`.
- **O Whisper "estica" as palavras por cima dos silêncios** e às vezes omite hesitações: medir só a distância entre palavras corta pouco. É preciso combinar com detecção de silêncio pela energia do áudio.
- **Haar cascade não acha rosto de perfil** em plano aberto. Usar YuNet.
- **Dessincronia de áudio/vídeo** com muitos cortes: alinhar todo início/fim de clipe à grade de quadros (30000/1001 fps) antes de montar o `trim`/`atrim`.
- Filtro muito grande: gravar em arquivo e usar `-/filter_complex filtro.txt` (sintaxe do FFmpeg 7; `-filter_complex_script` está obsoleto).
- Saída do Python com acentos no console do Windows: rodar com `PYTHONIOENCODING=utf-8`.

---

## Especificação: `scripts/transcrever.py`
Uso: `.venv/Scripts/python scripts/transcrever.py brutos/VIDEO.ext [modelo]`
- Modelo padrão `large-v3-turbo`, `device="cuda"`, `compute_type="float16"` (30 min de vídeo ≈ 45 s numa RTX 4070; o 1º uso baixa o modelo, ~1,6 GB).
- `language="pt"`, `word_timestamps=True`, `vad_filter=True` (`min_silence_duration_ms=500`), `condition_on_previous_text=False` (evita loops de repetição), `beam_size=5`.
- `initial_prompt` com os **nomes próprios do vídeo** + hesitações de exemplo ("Então... é... hum, a gente, né, tá, assim."), para grafar nomes e manter os "é…"/"hum" na transcrição.
- Salva `transcricoes/<nome>.json`: `{"segmentos":[{"s","e","texto","palavras":[{"w","s","e","p"}]}]}` e `transcricoes/<nome>.txt` com `[mm:ss.ss - mm:ss.ss] texto` por segmento.

## Especificação: `scripts/gerar_reel.py`
Uso:
```
python scripts/gerar_reel.py --video brutos/VIDEO.ext --transcricao transcricoes/VIDEO.json \
  --trechos "853.5-888.6,1592.8-1602.6" \
  --titulo "LINHA 1\NLINHA 2" --nome "NOME SOBRENOME" --cargo "cargo" \
  --credito "Fonte: <autor> · <licença>" --saida shorts/<nome>.mp4
```
`--trechos` são intervalos em segundos do vídeo original; vários trechos são concatenados na ordem dada.

Etapas, para cada trecho:
1. **Palavras**: pegar as palavras com `s >= inicio-0.05` e `e <= fim+0.05`.
2. **Limpeza**: remover hesitações (`hã, hum, hmm, ahn, ãh, eh, uh, ah, éé…`), "é" com duração ≥ 0,5 s isolado por pausas > 0,15 s dos dois lados, e gaguejo (mesma palavra repetida em seguida, intervalo < 0,6 s, fica só a última). Registrar no log o que foi removido.
3. **Pedaços por palavra**: juntar palavras enquanto o intervalo entre elas for ≤ `GAP_MAX`; cada pedaço vai de `1ª palavra - PAD_IN` a `última palavra + PAD_OUT` (ou `PAD_OUT_FRASE` se a palavra termina com `. ? !`).
4. **Ajuste no silêncio**: calcular RMS do áudio em janelas de 10 ms (16 kHz mono) e mover cada início para o ponto de menor energia em [-0,08 s, +0,03 s] e cada fim para o de menor energia em [-0,04 s, +0,08 s]. Isso evita engolir sílaba e estalo.
5. **Silêncios do áudio**: dentro de cada pedaço, cortar todo trecho abaixo do limiar por ≥ `MIN_SIL`, deixando `PAD_SIL` de cada lado. Limiar adaptativo: `clip(mediana_dB_do_trecho - 10, -42, -30)`. Descartar sobras < `MIN_PEDACO`.
6. **Planos de câmera**: `ffmpeg ... -vf "scale=320:-2,select='gt(scene,0.30)',showinfo"` e ler `pts_time`. Dividir os pedaços nas trocas de câmera.
7. **Rastreamento de rosto por plano**: amostrar 2 quadros/s (um único ffmpeg com `fps=2,scale=960:540`, saída `rawvideo bgr24`), detectar com YuNet (entrada 960×540, score ≥ 0,6) e converter para coordenadas 1920×1080.
   - **Close** se a mediana da largura do maior rosto for ≥ 180 px: o alvo é o maior rosto.
   - **Plano aberto**: o alvo é o rosto à direita/esquerda/centro, conforme `ALVO_ABERTO`.
   - Sem rosto: usar o vídeo 16:9 inteiro sobre ele mesmo desfocado (`boxblur=30:3`, brilho -0,10).
   - Suavizar a trajetória (cx, cy, altura do rosto) com mediana móvel de 5 amostras.
8. **Enquadramento 9:16 por clipe**:
   - Close: altura do recorte = `1080/zoom`, rosto a 36% da altura.
   - Plano aberto ("close virtual"): altura base = `clamp(altura_rosto × 7,5, 760, 1080)`, punch-in suavizado (`1 + (zoom-1)×0,65`), rosto a 40% da altura.
   - Largura = altura × 9/16, limitada às bordas do quadro. O recorte se move linearmente da posição do rosto no início do clipe até a do fim: `crop=w:h:'x0+(x1-x0)*t/d':'y0+(y1-y0)*t/d'`. Depois `scale=1080:1920:flags=lanczos,setsar=1`.
   - Zoom alterna a cada corte entre `ZOOMS = [1.00, 1.18]` (continua alternando entre trechos).
9. **Grade de quadros**: para cada trecho, a entrada começa em `S0 = início_do_1º_clipe - 0.5`. Arredondar os limites de cada clipe para quadros inteiros relativos a S0 (`round((t-S0)*30000/1001)`) e descartar clipes com < 3 quadros.
10. **Legendas na linha do tempo final**: remapear cada palavra mantida para `saida = inicio_do_clipe_na_saida + (t - inicio_do_clipe_na_fonte)`.
11. **Render num comando só**: um `-ss S0 -t dur -i video` por trecho; `split`/`asplit` por entrada; por clipe, `trim`/`atrim` + `setpts/asetpts=PTS-STARTPTS` + enquadramento + `afade` de 8 ms na entrada e na saída; `concat=n=N:v=1:a=1`; `subtitles=legenda.ass:fontsdir=fontes,format=yuv420p`; áudio `loudnorm=I=-14:TP=-1.5:LRA=11,aresample=48000`.
    Encode: `libx264 -preset medium -crf 18 -profile:v high -pix_fmt yuv420p -r 30000/1001`, `aac 192k`, `-movflags +faststart`.
12. Gravar `log.txt` (o que foi removido, planos, duração bruta → final) em `shorts/_trabalho/`.

## Estilo aprovado pelo usuário (padrão — só mudar se ele pedir)
| Item | Valor |
|---|---|
| Ritmo | `GAP_MAX=0.28`, `PAD_IN=0.06`, `PAD_OUT=0.10`, `PAD_OUT_FRASE=0.18`, `MIN_SIL=0.25`, `PAD_SIL=0.07`, `MIN_PEDACO=0.15` |
| Zoom | alterna 1,00× / 1,18× a cada corte |
| Legenda | ASS 1080×1920, Arial Black 86 px, MAIÚSCULAS, branca, contorno preto 8 + sombra 4, centro em (540, 1330); máx. 3 palavras / 15 caracteres por tela; troca de tela em pontuação ou pausa > 0,25 s; tira `. , ; : …` e aspas |
| Palavra falada | amarelo `#FFD400` (ASS `&H0000D4FF&`), "pop" de 122% → 108% em 110 ms (`\fscx122\fscy122\t(0,110,\fscx108\fscy108)`), depois `{\r}` |
| Título-gancho | Arial Black 74 px, texto quase preto sobre caixa amarela (`BorderStyle=3`, `Outline=22`), em (540, 250), de 0 a 3,4 s, entra com pop 60%→106%→100% e sai com fade de 250 ms; 2 linhas curtas separadas por `\N` |
| Nome / cargo | Arial Black 62 px branco em (540, 1080) + Arial negrito 46 px amarelo em (540, 1150), de 0,4 a 4,2 s, com fade |
| Crédito | Arial 32 px branco quase opaco em (540, 160), o vídeo inteiro: "Fonte: <autor> · <licença>" |
| Correções na legenda | dicionário `CORRECOES` (grafias que o Whisper erra, ex.: nomes de povos, sobrenomes) |
| Duração alvo | 30–60 s por Reel |

## Como escolher trechos
- Ler o `.txt` inteiro e procurar: uma ideia completa e curiosa, explicável em 30–60 s, com uma frase de abertura forte e uma frase de fecho (efeito "uau", conclusão ou pergunta).
- Pode juntar dois trechos distantes quando um complementa o outro (ex.: a história + a frase de efeito que aparece mais tarde).
- Evitar começar no meio de frase ou depender de contexto anterior (ex.: "como eu disse…").
- Propor o título-gancho (curto, em forma de curiosidade ou promessa) e mostrar ao usuário os trechos com a minutagem do original (mm:ss) antes de renderizar todos.
- Para limites exatos, consultar o `.json` (tempo de cada palavra); terminar o trecho logo depois do fim da última palavra, para não pegar o começo da frase seguinte.

## Verificação obrigatória antes de mostrar
1. `ffprobe` na saída: 1080×1920, h264 + aac, duração esperada.
2. Grade de quadros do Reel (ffmpeg `-ss t -frames:v 1` em ~6–12 pontos + `tile`) e olhar a imagem: rosto enquadrado, legenda legível, título e nome sem cobrir o rosto.
3. Retranscrever o Reel com o Whisper e comparar com o texto original do trecho: nenhuma palavra pode ter sido cortada.
4. Abrir o vídeo no player (`Start-Process <arquivo>` no Windows) e informar ao usuário: arquivo, duração, trechos usados com a minutagem do original, o que foi cortado e limitações (ex.: nitidez do close virtual, que amplia ~2,3× a imagem).

## Publicação
- Respeitar a licença do vídeo original. Em CC BY: colocar autor, link do original, nome da licença e "editado (cortes, legendas, reenquadramento)" na descrição/legenda do post.
- No YouTube, agregar valor (edição, legendas, cortes) por causa da política de conteúdo reutilizado.
- O crédito na tela é complemento; a interface do Instagram pode cobrir o topo, então o crédito na descrição é obrigatório.
- Sugerir ao usuário um texto de legenda para o post junto com cada Reel.
