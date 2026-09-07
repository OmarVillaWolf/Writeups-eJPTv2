# Crear lista de usuarios

**Tags:** #AD #UsernameAnarchy

## 1. Crear lista de nombres

```bash
nvim fullNames.txt
```

Ejemplo:

```text
Fergus Smith
Shaun Coins
Hugo Bear
```

## 2. Generar usuarios

* [Username-Anarchy](https://github.com/urbanadventurer/username-anarchy)

```bash
./username-anarchy --input-file fullNames.txt --select-format flast,first.last,firstlast,firstl,f.last,last.first,lfirst,lastf,last,first,fmlast,firstmiddlelast > usernames.txt
```

## Formatos

| Formato | Ejemplo |
|---------|----------|
| first | fergus |
| last | smith |
| flast | fsmith |
| first.last | fergus.smith |
| firstlast | fergussmith |
| firstl | ferguss |
| f.last | f.smith |
| last.first | smith.fergus |
| lfirst | sfergus |
| lastf | smithf |
| fmlast | fgsmith |
| firstmiddlelast | fergusgabrielsmith |

> **Tip:** Si identificas el formato de la empresa (ej. `jsmith`), genera únicamente ese formato.