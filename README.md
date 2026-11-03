# <M21> · <¿Emitimos alerta de incendio forestal hoy?>

> **Hackatón IIA · UADE · 3.4.257.** Este README es la plantilla del grupo. En su fork, reemplacen todo lo que está entre `< >` y borren esta nota. Las bases están en `BASES.md` y su ficha en `problematicas/`.

**Integrantes:** <paloma deverill> · <lucila Guiñazu>
**IA declarada:** <gemini> <claude>
**Pista:** < Reglas >
**Publicación en LinkedIn:** <link>

---

## 1. Formulación y PEAS

**La pregunta.** <Para cada caso (un día en una zona serrana), ¿emitimos alerta roja de incendio? SÍ significa: ese día se inicia un incendio que se propaga. NO significa: ese día no se inicia ningún incendio que se propague.>

| PEAS | Su sistema |
|---|---|
| Medida de rendimiento | Que no haya días con incendio sin alerta (sensibilidad), cuidando que no se movilicen brigadas de más (precisión y especificidad). |
| Entorno | <Zona serrana, una decisión por día y por zona, tomada cada mañana por el servicio de manejo del fuego.> |
| Actuadores | <Emitir o no emitir la alerta roja (movilizar brigadas, suspender actividades) |
| Sensores | <Humedad relativa (%), viento (km/h) y días sin lluvia, medidos antes de decidir.> |

**Objetivo y métrica.** <Qué es lo que de verdad importa en su problema y con qué métrica lo van a medir. ¿Son lo mismo?> <El objetivo es proteger hectáreas, fauna y viviendas: que no haya un incendio sin alerta previa. La métrica principal es la *sensibilidad* (de los días con incendio, cuántos alertamos). No es lo mismo que el objetivo: la sensibilidad sola se maximiza alertando todos los días, por eso la acompañamos con precisión y especificidad, que miden el costo de las falsas alarmas. No usamos la exactitud como métrica principal porque en estos datos hay más días sin incendio que con incendio y un sistema que siempre dice NO ya saca una exactitud alta sin proteger a nadie.>

## 2. El sistema

**Cómo decide.** <Sistema de reglas si-entonces. Se emite alerta (SÍ) si se cumple *al menos una* de estas reglas:

1. *R1 · Viento muy fuerte:* viento ≥ 43 km/h.
2. *R2 · Aire seco con viento:* humedad ≤ 30 % *y* viento ≥ 19 km/h.
3. *R3 · Sequía larga:* días sin lluvia ≥ 39.

Si no se cumple ninguna, la salida es NO.>

**Variables que usa y por qué.** <Usa las tres variables de la ficha, porque cada regla usa una distinta: humedad (el aire seco seca el combustible), viento (propaga el fuego) y días sin lluvia (acumulan sequedad). Los umbrales se fijaron con los casos de entrenamiento (1 a 14), en el punto medio entre el valor más extremo de un caso NO y el de un caso SÍ: en R1, entre 42 (caso 1, NO) y 44 (caso 12, SÍ); en R3, entre 34 (caso 5, NO) y 43 (caso 12, SÍ); en R2, el único caso NO con humedad baja es el 11 (humedad 27 %, viento 6 km/h), por eso la regla exige además viento: el umbral de 19 km/h queda entre 6 (caso 11, NO) y 31 (caso 2, SÍ), y el de humedad en 30 % cubre a todos los casos SÍ con aire seco (humedad máxima 26 %, caso 7). Usa solo datos que se conocen a la mañana, antes de decidir; la columna resultado_real nunca entra a las reglas.>

**Cómo se corre en vivo.** <A mano, con tres comparaciones:

1. ¿Viento ≥ 43? Si sí, SÍ.
2. Si no: ¿humedad ≤ 30 y viento ≥ 19? Si sí, SÍ.
3. Si no: ¿días sin lluvia ≥ 39? Si sí, SÍ.
4. Si ninguna se cumple, NO.

O en Python:

python
def alerta(humedad, viento, dias_sin_lluvia):
    return viento >= 43 or (humedad <= 30 and viento >= 19) or dias_sin_lluvia >= 39>

**Dos casos de ejemplo.**

| Caso inventado | Valores | Salida del sistema |
|---|---|---|
| A | <humedad 20 %, viento 35 km/h, 5 días sin lluvia> | <SÍ (R2: aire seco con viento)> |
| B | <humedad 60 %, viento 20 km/h, 12 días sin lluvia > | <NO (no cumple ninguna regla)> |

## 3. Medición

**Entrenamiento (casos 1–14)**

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = <6> | FN = <0> |
| **Real NO** | FP = <0> | VN = <8> |

**Prueba (casos 15–20)**

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = <3> | FN = <0> |
| **Real NO** | FP = <1> | VN = <2> |

| Métrica | Entrenamiento | Prueba |
|---|---|---|
| Exactitud | <100%> | <83,3%> |
| Precisión | <100%> | <75%> |
| Sensibilidad | <100%> | <100%> |
| Especificidad | <100%> | <66,7%> |

El único error está en la prueba: el caso 16 (humedad 24 %, viento 24 km/h, 21 días sin lluvia) es un falso positivo, porque cumple R2 y no hubo incendio.

**Entrenamiento contra prueba.** <¿Hay diferencia? ¿Por qué? ¿Sobreajuste o generalización?>
Entrenamiento contra prueba. Hay diferencia: 100 % contra 83,3 % de exactitud. El sistema se armó mirando los casos 1 a 14, así que ahí acierta todo; en casos nuevos aparece un error. Es una señal de sobreajuste moderado: con solo 14 casos de entrenamiento (6 con incendio) los umbrales quedan muy pegados a esos datos. Además, la prueba tiene solo 6 casos, así que un solo error mueve la exactitud 16,7 puntos y no se puede concluir mucho. Importante: la regla R1 se agregó después de ver que el sistema sin ella fallaba en el caso 18 (ver sección 4), por lo que la prueba ya no es completamente independiente. Sin R1, el resultado en prueba era VP = 2, FN = 1, FP = 1, VN = 2 (exactitud, precisión, sensibilidad y especificidad de 66,7 %).

## 4. El error más caro

<¿Es peor el falso positivo o el falso negativo en su problema? ¿Por qué, según la ficha? ¿Qué cambiaron en el sistema para cometer menos ese error?>
Según la ficha, el error más caro es el falso negativo: no alertar y que haya incendio significa hectáreas quemadas, fauna perdida y riesgo para viviendas. El falso positivo cuesta brigadas movilizadas de más y actividades suspendidas, un costo real pero recuperable.

Qué cambiamos por él: la primera versión del sistema (solo R2 y R3) tenía un falso negativo en la prueba, el caso 18 (humedad 57 %, viento 47 km/h, 26 días sin lluvia: viento fuerte con humedad moderada). Revisando el entrenamiento vimos que todos los días con viento ≥ 43 km/h habían sido incendio (casos 7, 10, 12 y 14), así que agregamos R1. Con eso la sensibilidad en prueba subió de 66,7 % a 100 %. El costo fue aceptar que R1 se agregó después de mirar la prueba, y lo declaramos arriba.

## 5. ¿Comparado contra qué?

**Línea base trivial.** <Qué resultado saca un sistema que responde siempre lo mismo, y cómo se compara el suyo.>

| Checklist (Clase 6) | Respuesta |
|---|---|
| ¿Qué datos? | <> |
| ¿Qué métrica? | <> |
| ¿Comparado contra qué? | <> |
| ¿Quién lo midió? | <> |

## 6. Riesgo: el camino completo

<Datos → modelo → decisión → consecuencia aplicado a su caso. Al menos un riesgo concreto.>

## 7. Bitácora

**Uso de la IA**

| # | Qué le pedimos | Qué aceptamos | Qué rechazamos y por qué |
|---|---|---|---|
| 1 | <> | <> | <> |

**Corridas contra el evaluador**

| Corrida | Nota | Agujero que encontró | Qué cambiamos |
|---|---|---|---|
| 1 | <> | <> | <> |
