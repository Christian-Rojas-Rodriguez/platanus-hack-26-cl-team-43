# Propuesta: harness de red team abierto + hardening verificado

> **Borrador para discutir con el equipo.** No hay código todavía: esto es solo la idea y los diagramas para decidir si la tomamos.

Un modelo **open-weight** ataca código generado por IA dentro de un sandbox aislado; un **oráculo determinístico** confirma cada exploit; un modelo **grande y cerrado** propone el arreglo; se vuelve a atacar para certificar que la falla murió sin romper los tests.

---

## Sistema — alto nivel

```mermaid
flowchart LR
    repo["Repo generado por IA"] --> H
    subgraph H["Harness"]
        direction LR
        red["Rojo<br/>open-weight local"] -->|intentos| orac["Oráculo<br/>determinístico"]
        orac -->|falla confirmada| blue["Azul<br/>modelo grande cerrado"]
        blue -->|git diff| orac
    end
    H -->|certificado| out["Falla 0 · tests 100%"]
```

**La asimetría es la idea:** el rojo es abierto porque los modelos cerrados rechazan payloads ofensivos; el azul es cerrado porque arreglar no dispara rechazos. El harness es la pieza fija; los dos modelos se enchufan.

---

## Sistema — bajo nivel

```mermaid
flowchart TB
    runner["runner · host<br/>orquesta y mide"]

    subgraph net["red sandbox — internal, sin internet"]
        target["target<br/>app bajo prueba<br/>non-root · read-only · cap_drop ALL"]
        redag["red-agent<br/>modelo open-weight"]
        oracle["oráculo<br/>chequeos + pytest"]
        witness["egress-witness<br/>registra toda salida"]
    end

    blue["blue-harness<br/>modelo grande cerrado<br/>fuera del sandbox"]

    runner -->|up / down efímero| net
    redag -->|ataca HTTP| target
    redag -.->|intento de salida| witness
    target -.->|canario de salida| witness
    oracle -->|verifica evidencia| target
    oracle -->|JSON resultado| runner
    runner -->|código + evidencia| blue
    blue -->|git diff| runner
    runner -->|aplica patch + rebuild| target
```

| Pieza | Rol | Red |
|---|---|---|
| `target` | app con la falla sembrada | sandbox |
| `red-agent` | genera los ataques | sandbox |
| `oráculo` | confirma con evidencia + corre tests | sandbox |
| `egress-witness` | atrapa cualquier salida a internet | sandbox |
| `blue-harness` | propone el arreglo | fuera |
| `runner` | ataca → arregla → re-ataca → mide | host |

Sandbox pensado como **Docker Compose**: red `internal` sin internet, contenedores efímeros, `target` endurecido (non-root, read-only, cap_drop).

---

## Casos de uso

```mermaid
flowchart LR
    evaluador(["👤 Evaluador / Lab"])
    equipo(["👤 Equipo de seguridad"])

    uc1["Atacar un repo en sandbox"]
    uc2["Confirmar exploits con evidencia"]
    uc3["Proponer y aplicar el arreglo"]
    uc4["Certificar: falla muerta + tests OK"]
    uc5["Verificar que el sandbox está cerrado"]
    uc6["Comparar modelos rojo/azul"]

    evaluador --> uc1
    evaluador --> uc4
    evaluador --> uc5
    equipo --> uc2
    equipo --> uc3
    equipo --> uc6
    uc1 -.->|incluye| uc2
    uc3 -.->|incluye| uc4
```

---

## Secuencia — el ciclo cerrado

```mermaid
sequenceDiagram
    autonumber
    participant R as runner
    participant Red as red-agent (abierto)
    participant T as target
    participant O as oráculo
    participant Blue as blue-harness (cerrado)

    R->>T: levanta app en sandbox
    R->>Red: dale los endpoints + código
    Red->>T: intentos de exploit
    T-->>O: respuestas observables
    O->>O: ¿canario leído? ¿egress? ¿test roto?
    O-->>R: falla CONFIRMADA + evidencia
    R->>Blue: falla + código afectado
    Blue-->>R: git diff
    R->>T: aplica patch + rebuild
    R->>Red: re-ataca (mismo exploit + mutaciones)
    T-->>O: respuestas
    O-->>R: exploit 0% · tests 100%
    Note over R: certificado ✔ (si no, reintenta hasta K)
```

---

## Secuencia — chequeo de contención (previo)

En 4 de 5 incidentes de evaluación de 2026 la causa fue una **red mal configurada** que nadie verificó antes de correr.

```mermaid
sequenceDiagram
    autonumber
    participant R as runner
    participant S as sandbox
    participant W as egress-witness

    R->>S: ¿red interna? ¿non-root? ¿sin docker.sock? ¿sin secretos?
    alt configuración floja
        S-->>R: ❌ falla el chequeo
        R->>R: frena — no corre el ataque
    else configuración correcta
        R->>S: canario hacia adentro (witness)
        W-->>R: ✔ registrado
        R->>S: canario hacia afuera (internet)
        S-->>R: ✔ bloqueado
        R->>R: sandbox cerrado — procede
    end
```

---

## Fases propuestas

```mermaid
flowchart LR
    f0["Fase 0<br/>sandbox + oráculo<br/>+ 1 fixture"] --> f1["Fase 1<br/>antes/después<br/>arreglo de referencia"]
    f1 --> f2["Fase 2<br/>chequeo de<br/>contención"]
    f2 --> f3["Fase 3<br/>modelos rojo + azul<br/>enchufados"]
    f3 --> f4["Fase 4<br/>demo + mutaciones"]
```

La **fase 1** ya es demo con la métrica completa (falla que baja + tests que pasan). Los modelos son el salto de wow, no un requisito para mostrar algo.

---

## Para decidir en equipo

- ¿Rojo corriendo de verdad (modelo abierto ofensivo) o chequeos fijos como piso? Las dos conviven.
- Modelos candidatos — rojo: WhiteRabbitNeo-7B, pentest-7b, foundation-sec-red-team; azul: el mejor cerrado por API, o Qwen2.5-Coder-7B local.
- Encaje con el track: cae en AI control + guardrails (ofensa). Línea roja a respetar: solo fixtures propios, nunca blancos externos.
- Métrica para el jurado: exploits confirmados antes vs. después, con tests al 100%.
