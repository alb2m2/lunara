# Skill: /sync-subs

Sincroniza subtítulos desincronizados usando ffsubsync — analiza el audio de la película y ajusta el `.srt` automáticamente.

## Uso

```
/sync-subs <título de la película o serie>
```

## Ejemplos

```
/sync-subs White Chicks
/sync-subs Breaking Bad S01E01
/sync-subs Dune Part Two
```

## Qué hace

1. Busca el archivo de video en `/mnt/data/media/Movies/` o `/mnt/data/media/TV/`
2. Encuentra el `.srt` asociado (`.es.srt`, `.en.srt`, etc.)
3. Corre `ffsubsync` — detecta el desfase usando el audio como referencia
4. Sobreescribe el `.srt` con los tiempos corregidos
5. Reporta el offset en segundos y el score de confianza

## Herramienta en el servidor

```bash
# Instalado en: /home/amatamoros/.local/bin/ffsubsync
# Requiere: ffmpeg (/usr/bin/ffmpeg)

~/.local/bin/ffsubsync "video.mkv" -i "subtitulo.srt" -o "subtitulo.srt"
```

## Instrucciones para Claude

1. Busca el archivo con `find /mnt/data/media -iname "*<título>*"` via `ssh matamoros-local`
2. Identifica el `.mkv` y los `.srt` disponibles — lista todos los idiomas encontrados
3. Corre ffsubsync sobre cada `.srt` encontrado
4. Reporta: offset detectado (segundos), score, y si fue exitoso
5. Si hay múltiples archivos de subtítulos (`.es.srt`, `.en.srt`), procésalos todos
6. Jellfyin toma los cambios automáticamente — no hay que reiniciar nada

## Notas

- Score > 100,000 = alta confianza
- Score < 10,000 = resultado dudoso, puede empeorar los subs
- Si el offset es 0.000 = ya estaban sincronizados
- El `.srt` original se sobreescribe — no hay backup automático
