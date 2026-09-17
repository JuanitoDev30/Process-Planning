# Bitácora: taller de planificación de procesos

## Primera parte: en papel

### Resolución a mano

Se resolvieron en papel los diagramas de Gantt de FIFO, SJF no expropiativo y
Round Robin con quantum 2, 1 y 8 (fotos en `capturas/`). Las tablas de tiempos
pasadas en limpio están en el [README](README.md).


### Punto 3: comparación de los tres algoritmos

**¿Cuál da el menor tiempo de espera promedio?**
SJF no expropiativo, con 4 unidades (frente a 4,75 de FIFO y 5 de Round Robin
con q = 2). Al llegar a t = 7 elige primero a P3, cuya ráfaga es 1, y así un
proceso muy corto no se queda esperando detrás de otros más largos.

**¿Se puede usar tal cual en un sistema real? ¿Por qué?**
No. SJF necesita saber de antemano cuánto va a durar la próxima ráfaga de CPU
de cada proceso, y el sistema operativo no tiene esa información: la duración
depende de los datos, de la entrada del usuario, de la E/S, etc. En el papel el
dato viene en la tabla, en la realidad no. Lo más que se puede hacer es
**estimar** la ráfaga siguiente a partir de las anteriores (por ejemplo, con
promedio exponencial), así que el resultado óptimo del papel no se consigue tal
cual.

Además tiene otros problemas prácticos:

- **Inanición:** si siguen llegando procesos cortos, un proceso largo puede no
  ejecutarse nunca.
- **No es expropiativo:** una vez que un proceso largo toma la CPU (como P1 aquí)
  no la suelta hasta terminar, lo que da malos tiempos de respuesta en sistemas
  interactivos.

### Punto 4: Round Robin con quantum 1 y quantum 8

**Quantum = 1** (espera promedio 5,5, retorno promedio 9,5)
Los procesos se van turnando en cada unidad de tiempo, así que el resultado se
parece a un **reparto equitativo del procesador**, como si cada proceso tuviera
una fracción de CPU para él solo y todos avanzaran "a la vez". P3, que es el más
corto, termina muy pronto (retorno 2), pero los largos se alargan y el promedio
empeora. Aquí no se cuenta el cambio de contexto; en un sistema real, con un
quantum tan chico habría muchísimos cambios de contexto (14 en este ejemplo) y
ese costo se comería buena parte del tiempo de CPU.

**Quantum = 8** (espera promedio 4,75, retorno promedio 8,75)
Como el quantum es mayor que la ráfaga más larga (7), ningún proceso es
interrumpido: cada uno se ejecuta completo en el orden en que llegó. El
resultado es **idéntico a FIFO**, con el mismo diagrama y los mismos tiempos.

**Conclusión:** Round Robin queda entre los dos extremos. Si el quantum es muy
grande, se convierte en FIFO; si es muy pequeño, se parece a compartir el
procesador por igual, pero el costo de los cambios de contexto se vuelve
excesivo. Un buen quantum debe ser mayor que la mayoría de las ráfagas cortas
sin llegar a ser tan grande que el algoritmo se vuelva FIFO.

---

## Segunda parte: en la máquina


### Punto 5: procesos en ejecución y prioridades

```
$ ps -eo pid,ni,pri,comm --sort=-pri | head
    PID  NI PRI COMMAND
     75  19   0 khugepaged
   3086  15   4 flatpak
    386  12   7 systemd-journal
    423  12   7 systemd-udevd
    813  12   7 systemd-resolve
    988  12   7 dbus-broker-lau
    993  12   7 NetworkManager
    995  12   7 accounts-daemon
    997  12   7 avahi-daemon
```

Observaciones:

- `NI` es el valor de amabilidad (*nice*), de −20 a 19. Cuanto **más alto**,
  **menos** prioridad: el proceso cede más el procesador.
- La columna `PRI` de `ps` está en escala invertida (un número mayor significa
  más prioridad), pero `--sort=-pri` ordena por el valor interno del núcleo, así
  que en la práctica deja arriba los de **menor** prioridad.
  Por eso encabeza la lista `khugepaged`, un hilo del núcleo que trabaja en
  segundo plano con nice 19.
- `top` además muestra en vivo el `%CPU`, el estado (`R` en ejecución, `S`
  dormido) y el tiempo de CPU acumulado (`HORA+` / `TIME+`).

### Punto 6: proceso que consume CPU y `renice`

Se lanzaron dos bucles infinitos fijados al mismo núcleo con `taskset`, para que
compitan por el procesador y el efecto de la amabilidad se note en el `%CPU`
(con un solo proceso en una máquina de 8 núcleos, este usaría el 100 % de su
núcleo con cualquier valor de *nice*).

```
$ taskset -c 0 bash -c 'while :; do :; done' &   # A
$ taskset -c 0 bash -c 'while :; do :; done' &   # B
$ top -p <A>,<B>
```

**Antes del renice:**

```
    PID USUARIO   PR  NI    VIRT    RES    SHR S  %CPU  %MEM     HORA+ ORDEN
 354287 walter    32  12   19764   3520   3264 R  50,0   0,0   0:02.10 bash
 354288 walter    32  12   19764   3444   3188 R  50,0   0,0   0:02.10 bash
```

**Se sube la amabilidad de B:**

```
$ renice -n 19 -p 354288
354288 (process ID) prioridad anterior 12, nueva prioridad 19
```

**Después del renice:**

```
    PID USUARIO   PR  NI    VIRT    RES    SHR S  %CPU  %MEM     HORA+ ORDEN
 354287 walter    32  12   19764   3520   3264 R  82,1   0,0   0:07.22 bash
 354288 walter    39  19   19764   3444   3188 R  17,9   0,0   0:03.20 bash
```

**Qué cambia:**

- `NI` de B pasa de 12 a 19 y `PR` de 32 a 39 (`PR = 20 + NI`).
- El reparto de CPU: antes era 50 % / 50 %; después, A sube a ~82 % y B baja a
  ~18 %. B se volvió "más amable" y cede el procesador a A.
- El tiempo acumulado (`HORA+`) de A empieza a crecer mucho más rápido que el
  de B.

**Otra observación:** en un primer intento se probó `renice -n 10` sobre un
proceso que ya tenía nice 12, y falló:

```
renice: no se ha podido establecer la prioridad de 354141 (process ID): Permiso denegado
```

Pasar de 12 a 10 es **bajar** la amabilidad, o sea, pedir más prioridad, y eso
solo lo puede hacer root. Un usuario normal solo puede subir el *nice* de sus
procesos (hacerlos más amables), nunca bajarlo.

Error frecuente evitado: un valor de amabilidad más alto **no** significa más
prioridad; es al revés.
