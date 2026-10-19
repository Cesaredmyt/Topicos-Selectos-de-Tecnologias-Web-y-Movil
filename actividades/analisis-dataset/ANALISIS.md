# Análisis del dataset de medios de transporte

Nombre: Cesar Enrique Diaz Maldonado
Matrícula: 22121331
Equipo:

---

## Actividad 1 — El retrato del dataset

Código: `actividad1_retrato.py`. Salida completa en `salida_actividad1.txt`.

### Predicción

1. Cuando llueve creo que domina la camioneta, para que no se moje el paquete y tampoco el conductor de la bici o de la moto. El dron sería imposible.
2. Para el centro creo que domina la moto, por la practicidad y porque no esperaría tanto en el tráfico como la camioneta. También puede ser la bicicleta, pero depende del volumen del paquete. El dron no va al centro porque no está despejado.
3. La distancia más larga de una bicicleta creo que es de 7 km, porque ese límite aparece en la tabla de reglas del README. Es una predicción que viene de una regla que ya conocía, no de deducirla.
4. No creo que haya envíos con una etiqueta que rompa las reglas del negocio, porque aunque el modelo prediga algo siempre se deben seguir las reglas, y si las reglas dicen que es imposible no se hace.

### Resultado

Porcentaje de cada medio según el clima:

| clima         | bicicleta | camioneta | dron | moto |
| ------------- | --------- | --------- | ---- | ---- |
| despejado     | 32.8      | 16.6      | 13.5 | 37.1 |
| lluvia        | 8.5       | 16.7      | 0.0  | 74.8 |
| viento_fuerte | 31.7      | 18.4      | 0.0  | 49.9 |

Porcentaje de cada medio según la zona de destino:

| zona        | bicicleta | camioneta | dron | moto |
| ----------- | --------- | --------- | ---- | ---- |
| centro      | 58.4      | 13.5      | 0.0  | 28.0 |
| periferia   | 0.9       | 14.0      | 12.6 | 72.5 |
| residencial | 26.2      | 14.6      | 17.3 | 41.9 |
| rural       | 0.0       | 40.8      | 0.0  | 59.2 |

Máximos por medio:

| medio     | peso_kg | volumen_l | distancia_km |
| --------- | ------- | --------- | ------------ |
| bicicleta | 8.0     | 39.7      | 7.0          |
| camioneta | 80.0    | 400.6     | 59.1         |
| dron      | 2.5     | 12.0      | 15.0         |
| moto      | 24.9    | 90.0      | 39.9         |

Etiquetas que rompen las reglas del negocio (con `es_posible`): **0 de 10 000**.

### Comparación

1. **Fallé.** Con lluvia domina la moto (74.8 %), no la camioneta (16.7 %). La camioneta tiene el mismo porcentaje con lluvia que despejado (16.6 %). No había considerado que la camioneta es la más cara y solo se usa cuando el paquete es pesado, grande, refrigerado o va muy lejos, y eso no cambia con la lluvia. Lo que hace la lluvia es frenar a la bicicleta (baja de 32.8 % a 8.5 %), y esos envíos pasan a la moto. Lo del dron sí lo acerté, con lluvia es 0 %.
2. **Casi.** En el centro gana la bicicleta (58.4 %) y la moto es segunda (28.0 %). El dron es 0 %, pero no por el clima, sino porque el centro es espacio aéreo restringido. Además en el centro las distancias son cortas (máximo 5.9 km), por eso la bicicleta gana.
3. **Acerté, pero porque ya conocía la regla.** El máximo de la bicicleta en el CSV es exactamente 7.0 km, igual que el límite del README. Lo que esto comprueba es que los datos sí respetan esa regla (ninguna bicicleta pasa de 7 km), no que yo la haya deducido.
4. **Acerté.** No hay ninguna etiqueta que rompa las reglas. Pero la razón no es que el modelo respete las reglas: este dataset no lo hizo el modelo, se generó aplicando las reglas primero.

### Preguntas

**¿En qué zonas y con qué climas el dron nunca aparece? ¿Qué regla lo explica?**
En lluvia y en viento fuerte, por la regla de que el dron solo vuela con clima despejado. En el centro, por la regla de espacio aéreo restringido. En zona rural tampoco aparece, pero ahí no es por una regla de zona: las distancias en rural empiezan en 15.9 km y el dron llega máximo a 15 km.

**La bicicleta casi no va a la periferia, aunque ninguna regla se lo prohíbe. ¿Qué otra columna lo explica?**
La distancia. Solo el 2.2 % de los envíos de periferia son de 7 km o menos (su mediana es 15.8 km) y la bicicleta llega máximo a 7 km. Por eso hay solo 18 bicicletas en periferia.

**Comparen los máximos con la tabla LIMITES. ¿Qué límites se pueden adivinar y cuál no?**
Los de bicicleta (8 kg, 40 L, 7 km), moto (25 kg, 90 L, 40 km) y dron (2.5 kg, 12 L, 15 km) se adivinan casi exactos con los máximos. El de la camioneta no: en los datos llega a 80 kg, 400 L y 59 km, pero el límite real es 1000 kg, 3000 L y 1000 km. Nadie manda un paquete tan grande, entonces los datos no lo muestran.

**Más de cien camionetas para viajes de menos de 2 km. ¿Cuántas tienen una razón clara?**
Son 107 en total. 95 tienen al menos una razón clara: 47 por peso mayor a 25 kg, 61 por volumen mayor a 90 L y 36 por ser refrigeradas (algunas cumplen más de una). Las otras 12 no tienen razón clara. Mi explicación es que el dataset cambia el 3 % de las etiquetas por otro medio que también era posible, y la camioneta siempre es posible, así que esos 12 son ruido.

### Para pensar

Sí se pueden construir reglas a partir de los datos, porque los datos siguen un patrón y eso sirve como un comienzo para tener las pautas que necesitamos. La mayoría de las reglas sí se podrían sacar, y también tomando en cuenta cosas lógicas. Las que se vieron claras fueron los límites de cada medio, en km, peso y volumen.

Lo que no se puede sacar: el límite de la camioneta, porque nadie manda paquetes tan grandes y los datos no lo muestran; la regla de que la bicicleta no va a zona rural, porque ahí las distancias ya son mayores de 7 km y la distancia lo explica sola; y los 12 casos raros de camioneta, que son ruido y no una regla.

El riesgo de dejar que el modelo descubra las reglas es que no toma en cuenta cosas lógicas si no se lo decimos, no tiene esa capacidad. Puede aprender el ruido como si fuera una regla, por ejemplo mandar una camioneta a 1 km, o encontrar patrones que son casualidad y que no nos ayudan. Además el modelo solo da probabilidades y no garantiza cumplir una restricción como que el dron no vuela de noche, mientras que una regla escrita a mano se cumple siempre.

---

## Actividad 2 — La hora y el dron

### Predicción

1. A las 3:00 creo que no sale ningún dron (0), porque de noche no salen drones: lo dice la regla del dron en el README.
2. Creo que alrededor del 25 % de los envíos sale de noche (antes de las 6:00 o desde las 21:00). La noche son 9 horas de las 24 del día (9 / 24 ≈ 37.5 %), así que si los envíos se repartieran parejo saldría ese porcentaje. Pienso que mi 25 % queda por debajo porque de noche se hacen muchos menos envíos que de día, por el hecho de que es de noche.

### Resultado

Código: `actividad2_hora.py`. Gráfica: `actividad2_hora.png`.

Envíos por hora de salida y medio:

| hora | bicicleta | camioneta | dron | moto |
| ---- | --------- | --------- | ---- | ---- |
| 0    | 2         | 12        | 0    | 63   |
| 1    | 6         | 11        | 0    | 49   |
| 2    | 5         | 19        | 0    | 59   |
| 3    | 4         | 13        | 0    | 60   |
| 4    | 2         | 10        | 0    | 47   |
| 5    | 8         | 25        | 0    | 116  |
| 6    | 94        | 50        | 33   | 136  |
| 7    | 204       | 92        | 68   | 233  |
| 8    | 205       | 118       | 57   | 252  |
| 9    | 187       | 97        | 65   | 279  |
| 10   | 192       | 123       | 68   | 273  |
| 11   | 178       | 107       | 72   | 283  |
| 12   | 217       | 111       | 82   | 297  |
| 13   | 211       | 112       | 70   | 284  |
| 14   | 204       | 103       | 72   | 274  |
| 15   | 199       | 122       | 81   | 303  |
| 16   | 213       | 108       | 59   | 260  |
| 17   | 192       | 97        | 69   | 292  |
| 18   | 162       | 97        | 72   | 248  |
| 19   | 166       | 78        | 45   | 199  |
| 20   | 115       | 72        | 35   | 160  |
| 21   | 20        | 58        | 0    | 240  |
| 22   | 5         | 28        | 0    | 116  |
| 23   | 5         | 15        | 0    | 55   |

- Drones a las 3:00: **0**.
- Horas sin ningún dron: **0, 1, 2, 3, 4, 5, 21, 22, 23**.
- Envíos de noche (antes de las 6 o desde las 21): **1053 de 10 000 = 10.5 %**.

### Comparación

1. **Acerté.** A las 3:00 salen 0 drones, como predije. La razón (la regla del README) se sostiene: las horas sin ningún dron son 0, 1, 2, 3, 4, 5, 21, 22 y 23, que son exactamente las 9 horas de noche. De las 6 a las 20 sí hay drones en todas las horas (desde 33 a las 6:00 hasta 82 a las 12:00).
2. **Fallé.** Predije alrededor del 25 % y salió 10.5 % (1053 de 10 000), muy por debajo. Si los envíos se repartieran parejo, la noche sería 9 de 24 horas, es decir 37.5 %. Mi idea de que de noche se hacen menos envíos iba en la dirección correcta, pero fui muy optimista: se hacen mucho menos de lo que pensé. Por ejemplo, a las 3:00 salen 77 envíos en total (4 + 13 + 0 + 60), mientras que a las 12:00 salen 707 (217 + 111 + 82 + 297).

### Preguntas

**¿Hay horas del día sin ningún dron? ¿Coinciden exactamente con la regla?**
Sí. Las horas sin ningún dron son 0, 1, 2, 3, 4, 5, 21, 22 y 23: 9 horas, todas de noche, con 0 drones en cada una. Coinciden exactamente con la regla del README (el dron solo sale de 6:00 a 20:59), porque la coincidencia va en los dos sentidos: de noche no hay ningún dron, y de las 6 a las 20 ninguna hora queda vacía (la que menos tiene es la hora 6, con 33 drones).

**¿Cuántos envíos son de noche y de día? ¿Qué consecuencia tiene para lo que el modelo aprende de la noche?**
De noche hay 1053 envíos (10.5 %) y de día 8947 (89.5 %), así que los datos están desbalanceados hacia el día. De noche el dron tiene 0 ejemplos, y de los 1053 envíos 805 son moto (76.5 %), 191 camioneta y solo 57 bicicleta. El modelo ve pocos ejemplos de la noche, y casi todos de un solo medio.

**Entre las 20:00 y las 21:00 las bicis casi desaparecen y suben las motos. ¿Por qué?**
A las 20:00 hay 115 bicicletas y a las 21:00 solo 20; en las motos pasa lo contrario: de 160 suben a 240. La causa está en el README, en la parte de "Riesgos" del costo: la bicicleta recibe penalizaciones de noche, y la noche empieza a las 21:00. Con esa penalización la bicicleta deja de ser la opción más barata y esos envíos pasan a la moto.

**¿Por qué a un árbol le cuesta representar "de noche"? ¿Cuántas preguntas mínimas necesitaría?**
Un árbol solo pregunta cosas como "¿hora <= X?", y cada pregunta corta la línea de horas en dos tramos. La noche no es un tramo seguido: son las horas 0 a 5 y 21 a 23, y el día (6 a 20) queda en medio. Un solo corte no puede separarlas, así que necesita **mínimo 2 preguntas** (por ejemplo "hora <= 5" y "hora <= 20"). Lo comprobé con un árbol que solo ve `hora_salida` y debe adivinar si es de noche: con 1 nivel de preguntas acierta 94.9 % y con 2 niveles acierta 100 %. El 5.1 % que falla con 1 nivel son los 511 envíos de las horas 0 a 5, que quedan del lado equivocado.

### Para pensar

Mi idea es agregar una columna nueva llamada `es_noche`, que vale 1 si el envío sale de noche y 0 si no. Así, en vez de que el árbol tenga que hacer dos preguntas con la hora, le basta una: "¿es de noche?". Esta columna se calcula con la hora de salida que ya tenemos: es de noche si la hora es menor que 6 o es 21 o más (`(hora_salida < 6) | (hora_salida >= 21)`), que es la misma regla de noche que ya usa el programa para revisar si un medio es posible.

Para comprobar que ayudó, entrené el mismo árbol dos veces, una sin la columna nueva y otra con ella, y los puse a adivinar los mismos 2000 envíos de prueba (219 de ellos de noche). El código está en `actividad2_noche.py`.

| | sin `es_noche` | con `es_noche` |
| --- | --- | --- |
| Aciertos en todos los envíos | 92.1 % | 93.0 % |
| Aciertos en los envíos de noche | 81.7 % | 94.1 % |
| Errores en los 219 envíos de noche | 40 | 13 |
| Motos que el modelo mandó a dron de noche | 26 | 0 |

Con los resultados a la vista, la columna ayudó, y mucho de noche: los errores de noche bajaron de 40 a 13. Además, desaparecieron los 26 casos en que el modelo mandaba un dron a un envío de noche, algo que la regla del dron prohíbe. En el total la mejora se ve poca (de 92.1 % a 93.0 %) porque la noche es solo cerca del 11 % de los envíos, y ahí es donde estaba el problema. Esta prueba se hizo con una sola separación de los datos, así que para estar más seguro habría que repetirla con otras.

Por todo eso, mi decisión es que sí conviene agregar la columna. Es una sola línea de código, no le pide ningún dato nuevo a la app (sale de la hora) y quita justo el error más molesto: mandar un dron de noche.

## Actividad 3 — Anatomía de los errores

### Predicción

1. Creo que la confusión más frecuente será entre moto y bicicleta (sobre todo una moto que el modelo manda a bicicleta). Son los dos medios más parecidos: los dos llevan paquetes chicos a distancias cortas, y lo que los separa es el límite de la bicicleta (8 kg y 7 km). Un paquete que está cerca de ese límite es fácil de confundir.
2. Sobre si el modelo confunde alguna vez bicicleta con camioneta, creo que casi nunca o nunca. Son muy distintas: la bicicleta solo lleva paquetes chicos y cerca, y la camioneta se usa para lo pesado, lo muy grande, lo lejano o lo refrigerado. Nada se parece entre los dos.

### Resultado

Código: `actividad3_errores.py`. Salida completa en `salida_actividad3.txt`. Se usó el modelo guardado (`modelo_medios.joblib`) con la misma separación de datos de la guía (`test_size=0.2`, `stratify`, `random_state=42`): 2000 envíos de prueba.

De los 2000 envíos de prueba, el modelo se equivocó en **158 (7.9 %)** y acertó 92.1 %.

Cuántas veces pasó cada error (real → lo que dijo el modelo):

| real      | el modelo dijo | veces |
| --------- | -------------- | ----- |
| moto      | bicicleta      | 46    |
| camioneta | moto           | 32    |
| moto      | dron           | 29    |
| bicicleta | moto           | 20    |
| dron      | bicicleta      | 18    |
| camioneta | bicicleta      | 9     |
| camioneta | dron           | 4     |

Tabla completa (filas: lo real, columnas: lo que dijo el modelo):

| real \ dijo | bicicleta | camioneta | dron | moto |
| ----------- | --------- | --------- | ---- | ---- |
| bicicleta   | 539       | 0         | 0    | 20   |
| camioneta   | 9         | 291       | 4    | 32   |
| dron        | 18        | 0         | 172  | 0    |
| moto        | 46        | 0         | 29   | 840  |

De las 29 motos que el modelo mandó a dron, 26 son de noche, y para las 26 un dron es imposible según las reglas del negocio (la única razón en todas es la noche). Las otras 3 no se corrigen con las reglas, porque un dron sí era posible para ellas: salen de día (13, 17 y 18 h), con clima despejado, a zona residencial, pesan entre 0.9 y 1.67 kg, miden entre 3.7 y 6.6 L, valen entre 316 y 456 pesos, no son frágiles y tienen tráfico alto. Su distancia es de 7.35, 7.55 y 12.14 km.

De las 46 motos que el modelo mandó a bicicleta, la distancia va de 1.4 a 6.9 km (mediana 3.9 km): solo 5 están entre 6 y 7 km y las otras 41 están por debajo de 6 km. Solo 2 de las 46 son imposibles para una bicicleta según las reglas. Al comparar las medianas con las de las bicicletas que el modelo sí acertó, casi no hay diferencia: distancia 3.88 vs 3.37 km, peso 1.81 vs 1.69 kg, volumen 9.05 vs 7.32 L, valor 872 vs 868 pesos y hora 13.5 vs 13. Lo que sí cambia es la prioridad y el clima:

| | motos mandadas a bici | bicis acertadas |
| --- | --- | --- |
| prioridad `mismo_dia` | 48 % | 12 % |
| prioridad `normal` | 22 % | 58 % |
| clima despejado | 52 % | 85 % |
| clima lluvia | 24 % | 2 % |
| clima viento fuerte | 24 % | 13 % |
| sale de noche | 11 % | 2 % |

Las camionetas que el modelo manda a bicicleta son 9, y todas son viajes de 1.66 a 5.06 km, ligeros (0.61 a 4.88 kg) y ninguno es refrigerado. Para las 9, una bicicleta (y una moto) es posible según las reglas, así que las reglas no corrigen ninguna.

Al usar otro `random_state` (el mismo árbol, pero con otra separación de los datos), salió esto:

| random_state | errores | exactitud | error #1 | error #2 |
| --- | --- | --- | --- | --- |
| 1 | 161 | 92.0 % | moto → bicicleta (37) | camioneta → moto (29) |
| 7 | 149 | 92.6 % | dron → bicicleta (37) | moto → dron (26) |
| 42 | 158 | 92.1 % | moto → bicicleta (46) | camioneta → moto (32) |
| 99 | 150 | 92.5 % | dron → bicicleta (35) | camioneta → moto (29) |
| 2024 | 151 | 92.4 % | moto → bicicleta (32) | camioneta → moto (32) |

Al cambiar de modelo (un bosque de 200 árboles en vez de un solo árbol, con la misma separación), los errores bajaron de 158 a 138 (exactitud 92.1 % → 93.1 %). Moto → bicicleta baja de 46 a 38, y moto → dron baja de 29 a 6.

### Comparación

1. **Acerté, pero la razón estaba a medias.** Predije que la confusión más frecuente sería moto con bicicleta, y salió moto → bicicleta con 46 de los 158 errores. Pero yo pensaba que era por paquetes cerca del límite de 7 km, y no es así: solo 5 de los 46 están entre 6 y 7 km, y la mediana es 3.9 km. Los datos muestran que se parecen en tamaño y distancia, pero lo que los separa es la prioridad y el clima (ver pregunta 3).
2. **Fallé en parte.** Dije que casi nunca o nunca habría confusión entre bicicleta y camioneta. Bicicleta → camioneta sí fue 0, pero camioneta → bicicleta pasó 9 veces. No había considerado que la camioneta también aparece en viajes cortos y ligeros sin ninguna razón clara, por el ruido que el README dice que se agregó a las etiquetas. Aquí lo confirman los datos: los 9 son viajes de menos de 5.1 km, ligeros y sin refrigerar, y 4 de ellos son los mismos casos raros de camioneta que ya vi en la actividad 1.

### Preguntas

**¿Cuántos errores hay y cuáles son las 3 confusiones más frecuentes? ¿Son entre medios parecidos o muy distintos? ¿Qué dice eso?**
Hay 158 errores de 2000 (7.9 %). Las 3 más frecuentes son moto → bicicleta (46), camioneta → moto (32) y moto → dron (29); juntas son 107 de los 158 errores (68 %). Son medios parecidos o vecinos: la moto está en las tres, y en todas el modelo se confunde con un medio que también serviría o que está justo al lado en tamaño. En cambio, entre los más distintos casi no hay errores: bicicleta → camioneta es 0, y dron ↔ camioneta es 4 en un sentido y 0 en el otro. Eso dice que el modelo aprendió bien los límites claros (peso, volumen, distancia, refrigerado) y se equivoca en las decisiones difíciles, donde dos medios son posibles y la diferencia depende del costo.

**De las motos que el modelo mandó a dron, ¿cuántas son de noche? ¿Cuáles corregirían las reglas del negocio y cuáles no? ¿Qué tienen en común las que no?**
De 29 motos mandadas a dron, 26 son de noche. Las reglas del negocio corrigen esas 26, porque el dron no puede salir de noche; quedan 3 que no corrigen. Esas 3 tienen en común que son envíos de día, con clima despejado, a zona residencial, ligeros (0.9 a 1.67 kg), pequeños (3.7 a 6.6 L), baratos (316 a 456 pesos), no frágiles y con tráfico alto. Un dron sí era posible para ellos, así que no hay ninguna regla que los marque como error: son casos donde dron y moto eran opciones válidas y casi parejas en costo.

**De las motos que el modelo mandó a bicicleta, ¿están cerca del límite de 7 km? Si no, ¿qué columnas las distinguen de las bicicletas bien acertadas?**
No están cerca del límite. De las 46, solo 5 están entre 6 y 7 km; las otras 41 están por debajo de 6 km, con una mediana de 3.9 km. En distancia, peso, volumen y valor casi no se distinguen de las bicicletas acertadas (por ejemplo 3.88 contra 3.37 km de mediana). Las columnas que sí las distinguen son la prioridad (48 % son `mismo_dia`, contra 12 % en las bicicletas acertadas) y el clima (solo 52 % es despejado contra 85 %, y 24 % es lluvia contra 2 %). Es decir, son envíos urgentes o con mal clima, donde conviene la moto, que es más rápida, y el modelo los manda a bicicleta por ser pequeños y cercanos.

**¿Qué pasaría con el análisis si usara otro `random_state`?**
Los números cambiarían un poco, porque cada `random_state` separa distintos 2000 envíos de prueba. Lo comprobé con 5 valores: los errores van de 149 a 161 y la exactitud de 92.0 % a 92.6 %, así que el tamaño del problema se mantiene. Pero el orden de las confusiones sí cambia: con `random_state=7` y `99` la confusión número 1 es dron → bicicleta (37 y 35) y no moto → bicicleta, y las motos mandadas a dron van de 20 a 29. Lo que se mantiene es el patrón (los errores están entre medios vecinos). Por eso conviene sacar conclusiones del patrón y no de un número exacto de una sola separación.

### Para pensar

Si solo pudiera arreglar un tipo de error, arreglaría moto → bicicleta (46 errores, 29 % del total). No elegiría moto → dron, porque de sus 29 errores 26 son de noche y las reglas del negocio ya los corrigen, así que solo quedan 3 y arreglarlos casi no cambia nada. Tampoco elegiría camioneta → bicicleta: son solo 9 y parecen ruido en las etiquetas, viajes cortos y ligeros que no tienen razón clara para ir en camioneta, así que ni el mejor modelo debería acertarlos. En cambio moto → bicicleta es el error más frecuente y las reglas casi no lo detectan: solo 2 de 46 son imposibles para una bicicleta, así que 44 llegarían al cliente como recomendación equivocada.

Y no lo arreglaría con reglas del negocio, porque en esos envíos tanto la moto como la bicicleta son posibles, y una regla solo prohíbe lo imposible, no decide entre dos opciones válidas. Tampoco parece un problema de datos faltantes, porque las columnas que los distinguen (prioridad y clima) ya existen en el dataset. Queda cambiar el modelo, o ayudarle a ver mejor la combinación de urgencia y clima.

Lo que ya comprobé sobre el cambio de modelo: un bosque de 200 árboles baja moto → bicicleta de 46 a 38 (solo 8 menos), y la exactitud sube de 92.1 % a 93.1 %. O sea, cambiar el modelo ayuda, pero poco para este error; por eso mi recomendación es probar primero darle al modelo una columna que junte prioridad y clima (por ejemplo "urgente y con mal clima") antes de pasar a un modelo más grande. Esa idea no la probé, es solo una propuesta. Lo que no se va a poder quitar es el ruido que el README dice que se agregó a propósito (3 % de etiquetas cambiadas y 8 % de variación en los costos): incluso el bosque se queda en 93.1 %.

## Actividad 4 — ¿Qué columnas usa de verdad el modelo?

### Predicción

Creo que el orden de más a menos importante es: distancia_km, peso_kg, volumen_l, refrigerado, prioridad, clima, hora_salida, trafico, valor_declarado_mxn, zona_destino, fragil, largo_cm, ancho_cm y alto_cm. Las que creo que el modelo no usa son largo, ancho y alto, porque el volumen ya las junta, y a lo mejor fragil. Mi razón es que las reglas del README dependen sobre todo de la distancia, el peso, el volumen y si va refrigerado (solo camioneta); lo demás solo mueve un poco el costo.

### Resultado

Código: `actividad4_columnas.py`. Salida completa en `salida_actividad4.txt`. Las columnas de categorías (clima, prioridad, zona, tráfico) vienen partidas en varias columnas de 0 y 1, así que sumé sus partes para tener las 14 columnas originales.

| columna | importancia | si la quito, la exactitud queda en | cambio |
| --- | --- | --- | --- |
| distancia_km | 32.20 % | 79.3 % | -12.80 |
| volumen_l | 19.29 % | 90.6 % | -1.50 |
| clima | 14.38 % | 84.4 % | -7.65 |
| refrigerado | 13.13 % | 87.8 % | -4.25 |
| peso_kg | 12.63 % | 90.4 % | -1.70 |
| hora_salida | 5.87 % | 89.1 % | -3.00 |
| fragil | 1.99 % | 91.6 % | -0.55 |
| prioridad | 0.31 % | 92.2 % | +0.05 |
| valor_declarado_mxn | 0.07 % | 92.1 % | 0.00 |
| ancho_cm | 0.07 % | 92.0 % | -0.05 |
| largo_cm | 0.03 % | 92.0 % | -0.05 |
| alto_cm | 0.03 % | 92.0 % | -0.05 |
| trafico | 0.02 % | 92.1 % | 0.00 |
| zona_destino | 0.00 % | 92.1 % | 0.00 |

Con las 14 columnas la exactitud es 92.1 %. Las primeras preguntas del árbol (paso 8) son: ¿distancia <= 7.01 km?, luego ¿volumen <= 90 L?, luego ¿está lloviendo? y después el peso o si es frágil. Para el tema de la pregunta 3: los envíos al centro son 2965, todos de 5.9 km o menos, y el modelo dijo dron en 0 de los 629 envíos de centro de la prueba. De los 948 drones, solo 83 (8.8 %) son de 5.9 km o menos, y fuera del centro, de los envíos de 5.9 km o menos solo el 5.9 % son dron (contra 15.4 % en los más largos).

### Comparación

Acerté en lo más importante: la distancia es la número 1 y refrigerado quedó entre las 4 primeras. Fallé en tres cosas. El clima lo puse en el lugar 6 y quedó tercero (14.4 %, y quitarlo baja 7.65 puntos), no había pensado que la lluvia le quita la bicicleta a muchos envíos. El peso lo puse segundo y quedó quinto. Y pensé que largo, ancho, alto y fragil no se usaban, pero solo zona_destino tiene importancia 0; las medidas casi no cuentan (0.03 a 0.07 %), pero fragil sí se usa (1.99 %). También me equivoqué con la prioridad: la puse quinta y casi no pesa (0.31 %).

### Preguntas

Las 4 columnas más importantes son distancia (32.2 %), volumen (19.3 %), clima (14.4 %) y refrigerado (13.1 %). Coinciden con las primeras preguntas del árbol, que son distancia, volumen y lluvia, y luego peso (refrigerado aparece más abajo en la rama larga). Con mi predicción coincidieron dos de las cuatro: distancia y refrigerado.

Al quitar una columna, la que más baja la exactitud es la distancia, de 92.1 % a 79.3 % (12.8 puntos menos), y le siguen el clima (7.65 menos) y refrigerado (4.25 menos). No cambian la exactitud la zona, el valor declarado y el tráfico (0.00), la prioridad (sube 0.05) y las medidas largo, ancho y alto (bajan 0.05, que es 1 envío de 2000). Peso y volumen tienen importancia alta pero quitar uno solo baja poco (1.7 y 1.5 puntos): la importancia no es lo mismo que lo que pierdo si la quito, porque otras columnas pueden cubrir su lugar, como las tres medidas que dan el volumen.

La zona de destino tiene importancia cero porque el árbol nunca la usa para preguntar, y aun así el dron no llega al centro por dos razones. La primera es que los envíos del centro son cortos (todos de 5.9 km o menos, mediana 2.8 km) y el dron casi no se usa en trayectos cortos, así que el modelo aprende con la distancia a mandar esos envíos a bicicleta o moto: en la prueba dijo dron en 0 de 629 envíos de centro. La distancia no dice la zona por sí sola (de 0 a 6 km, 67 % son centro y 33 % residencial), pero para el dron alcanza. La segunda es que, aunque el modelo dijera dron, la plataforma revisa las reglas (`es_posible`) y descarta el dron en el centro.

### Para pensar

Dejaría de pedirle al modelo el valor declarado y el tráfico, porque quitarlos no cambia la exactitud (0.00 puntos). Las medidas largo, ancho y alto tampoco aportan al modelo una vez que existe el volumen (cambian 0.05 puntos), aunque sí hay que pedirlas para calcular el volumen. Pero la zona sí hay que seguir pidiéndola aunque el modelo no la use: las reglas del negocio la necesitan para prohibir el dron en el centro y la bicicleta en zona rural. La prioridad y el valor pueden servir para cobrar, para el seguro o para decidir qué se entrega primero, aunque el modelo no los use. Mi postura es que sí conviene dejar de pedir los datos que solo estorban, pero antes hay que revisar para qué más se usan en la app, y no quitar nada solo porque el modelo lo ignore.

## Actividad 5 — El dron es minoría: ¿hay que balancear el dataset?

### Parte A: pensar antes de tocar código

Si hubiera un "modelo" que nunca dice dron pero acierta todo lo demás, tendría 1810 aciertos de 2000, o sea 90.5 %. Uno que siempre dice moto acertaría 915 de 2000, o sea 45.8 %. El primero suena muy bien y nunca encuentra un dron, así que la exactitud engaña cuando una clase es chica. Mejor usar la precisión (de los que el modelo dijo dron, cuántos sí lo eran) y el recall (de los drones reales, cuántos encontró), que salen en el reporte del paso 7.

Hay pocos drones porque así es la empresa, no por un error en los datos: el README dice que es a propósito, porque el dron solo se puede con clima despejado, de 6:00 a 20:59, fuera del centro y para paquetes de hasta 2.5 kg, 12 L y 15 km. Eso sí cambia la decisión: si en la realidad solo el 9.5 % de los envíos son dron, no conviene "inventar" más drones, porque el modelo se acostumbraría a una empresa que no existe.

Si balanceo, lo haría después de separar entrenamiento y prueba, y solo el entrenamiento. La prueba no se balancea, porque tiene que parecerse a los envíos reales que van a llegar. Si balanceo antes de separar, copias del mismo envío caen en entrenamiento y en prueba, y el modelo ya vio las respuestas del examen.

### Parte B: resultados

Código: `actividad5_balanceo.py`. Salida completa en `salida_actividad5.txt`. Todas las estrategias entrenan un árbol igual al del paso 6 y se miden con los mismos 2000 envíos de prueba (190 drones). Para la estrategia 5 usé `generar_dataset.py` (lo copié de tu carpeta de Descargas) con 40 000 filas y la semilla 7, y de ahí tomé solo drones nuevos para el entrenamiento; con la semilla por defecto (42) ese generador da tu mismo CSV, así que es el mismo.

| estrategia | envíos de entrenamiento | exactitud | precision dron | recall dron | veces que dice dron | sugerencias imposibles | drones de más que las reglas no corrigen |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Sin balancear | 8000 | 92.1 % | 0.839 | 0.905 | 205 | 31 | 7 |
| 1. Quitar sobrantes | 3032 | 89.0 % | 0.859 | 0.932 | 206 | 19 | 15 |
| 2. Repetir solo drones | 10905 | 90.0 % | 0.728 | 1.000 | 261 | 54 | 37 |
| 3. Repetir todas las chicas | 14652 | 90.2 % | 0.766 | 1.000 | 248 | 35 | 37 |
| 4. `class_weight="balanced"` | 8000 | 90.2 % | 0.766 | 1.000 | 248 | 35 | 37 |
| 5. Drones nuevos del generador | 10905 | 90.0 % | 0.722 | 1.000 | 263 | 56 | 37 |

Los drones de más son los que el modelo dijo dron y no lo eran: 33 sin balancear, 29, 71, 58, 58 y 73 en las otras. A la última columna le quité los que las reglas del negocio ya corrigen (un dron de noche, con lluvia, etc.).

La trampa que hice a propósito fue balancear antes de separar: repetí los drones de todo el dataset hasta igualar a la moto y después separé 80/20.

| | honesto (sin balancear) | trampa (balanceo antes) |
| --- | --- | --- |
| drones en la prueba | 190 | 916 |
| precision dron | 0.839 | 0.924 |
| recall dron | 0.905 | 0.989 |
| veces que dice dron | 205 | 981 |

Con la trampa los números mejoran, pero de los 916 drones de la prueba, 902 ya estaban (idénticos) en el entrenamiento, y en total 902 de 2726 envíos de prueba (33.1 %) eran copias de envíos que el modelo ya había visto. La mejora es falsa porque el modelo no adivinó esos drones, los recordó. Por eso la prueba tiene que apartarse antes de repetir o quitar nada.

### Parte C: preguntas

Con las estrategias que repiten drones o dan más peso (2, 3, 4 y 5), el recall del dron sube de 0.905 a 1.000, o sea que encuentra los 190 drones, pero la precision baja de 0.839 a entre 0.722 y 0.766. Una sube y la otra baja porque, al ver más drones, el modelo se anima a decir dron en casos dudosos: atrapa más drones de verdad, pero también más falsos (los drones de más pasan de 33 a entre 58 y 73). Es como una red de pesca más grande, que agarra más peces pero también más basura.

La estrategia de quitar sobrantes dejó solo 3032 envíos de los 8000 (se tiraron casi 5000 envíos de las otras clases) y fue la de menor exactitud, 89.0 %, 3.1 puntos menos que sin balancear. Se pierde información de los demás medios para poder igualar al dron, que es el más chico.

La 3 y la 4 dieron exactamente los mismos números (precision 0.766, recall 1.000, 248 veces dron, 35 imposibles), y la 2 y la 5 casi lo mismo (0.728 contra 0.722 de precision, 261 contra 263 veces dron). Son la misma idea: lo que importa es qué tan seguido ve el árbol al dron al entrenar, no si los drones son copias, nuevos o solo cuentan más. Repetir todas las clases chicas hasta igualar a la moto y usar `class_weight="balanced"` le dan al árbol la misma cuenta de ejemplos (repetir una copia pesa lo mismo que contarla más veces), y repetir drones o traer drones nuevos deja la misma proporción de drones (33.6 %), así que el árbol aprende casi igual.

El motivo "confianza 84 %" ya no significa lo mismo después de balancear. Esa confianza sale de contar cuántos ejemplos de cada medio había en la hoja del árbol, y si al entrenar el dron pasó de 9.5 % de los envíos a 25 % o 33.6 %, cada dron pesa más en esa cuenta, aunque en la realidad siga siendo 9.5 % (190 de 2000). Se nota en que la probabilidad media de dron sube de 0.102 sin balancear a 0.124 con drones repetidos y el modelo dice dron 261 veces en vez de 205, cuando los drones reales son 190. Entonces el 84 % deja de ser "de cada 100 envíos parecidos, 84 fueron dron en la vida real" y pasa a ser un número inflado.

### Parte D: decisión

Le recomendaría a la empresa no usar el modelo balanceado y dejar el original. Los números lo respaldan: balancear sí encuentra los 18 drones que el original se salta (el recall pasa de 0.905 a 1.000), pero a cambio mete 25 drones de más, y de esos las reglas del negocio solo corrigen 21 en la estrategia 3 y 4, así que los drones equivocados que llegan al cliente suben de 7 a 37. Además la exactitud baja de 92.1 % a 90.2 % o menos, y la confianza que muestra la app se infla.

Sobre qué cuesta más: no usar un dron cuando convenía es un error barato, porque se manda otro medio que sí es posible (la moto), algo más caro o más lento. Un dron cuando no convenía puede ser peor, porque implica un paquete ligero con un valor o una fragilidad que el dron penaliza, y además hay pocos. Esto último es lo que veo en los datos del README (los costos del dron penalizan lo frágil y lo de valor alto), pero no tengo los costos reales de la empresa, así que si ellos dicen que perder un envío en dron cuesta muy poco, la recomendación podría cambiar.

Las reglas del negocio corrigen buena parte de los errores del dron (26 de 33 sin balancear), pero no los que son posibles pero no convenientes, que son los que balancear aumenta. Y si la empresa compra el doble de drones, el problema cambia: entonces los datos reales tendrían más drones y lo correcto es volver a entrenar con esos datos nuevos, no balancear a la fuerza, porque la proporción ya reflejaría la realidad. "No balancear" es válido hoy porque el 9.5 % es lo que pasa en la empresa.
