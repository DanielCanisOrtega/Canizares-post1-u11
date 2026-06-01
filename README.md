# Estado Inicial del Proyecto

El servicio `PedidoService` contenía múltiples problemas de diseño:

### Code Smells Detectados

| Smell                        | Descripción                                                                              |
| ---------------------------- | ---------------------------------------------------------------------------------------- |
| Long Method                  | `procesarPedido()` realizaba validación, cálculo, descuento, notificación y persistencia |
| Large Class                  | PedidoService concentraba demasiadas responsabilidades                                   |
| Primitive Obsession          | Exceso de parámetros primitivos relacionados con datos del cliente                       |
| Field Injection              | Uso de `@Autowired` sobre atributos                                                      |
| Alta Complejidad Ciclomática | Múltiples decisiones dentro del mismo método                                             |

---

# Métricas SonarQube — Antes de la Refactorización

| Métrica                                  | Valor  |
| ---------------------------------------- | ------ |
| Complejidad Ciclomática (procesarPedido) | 16     |
| Code Smells                              | 18     |
| Bugs                                     | 0      |
| Vulnerabilidades                         | 0      |
| Technical Debt Ratio (TDR)               | 6.8%   |
| Deuda Técnica Estimada                   | 2h 15m |
| Maintainability Rating                   | C      |

---

# Técnica 1 — Introducción de Value Objects

## Problema

El método recibía una gran cantidad de parámetros relacionados con información del cliente:

```java
Long clienteId,
String clienteNombre,
String clienteEmail,
String clienteTelefono,
String clienteDireccion,
String clienteCiudad,
String clienteCodigoPostal
```

Esto representa un caso clásico de Primitive Obsession y Data Clump.

## Solución

Se creó el Value Object `DatosCliente`.

```java
public class DatosCliente {

    private final String nombre;
    private final String email;
    private final String telefono;
    private final Direccion direccion;

    public DatosCliente(
            String nombre,
            String email,
            String telefono,
            Direccion direccion) {

        if(nombre == null || nombre.isBlank())
            throw new IllegalArgumentException();

        if(email == null || !email.contains("@"))
            throw new IllegalArgumentException();

        this.nombre = nombre;
        this.email = email;
        this.telefono = telefono;
        this.direccion = direccion;
    }
}
```

## Beneficios

* Encapsulación de datos.
* Validaciones centralizadas.
* Menor cantidad de parámetros.
* Mayor legibilidad.

---

# Técnica 2 — Extract Method

## Problema

`procesarPedido()` tenía más de 40 líneas y múltiples responsabilidades.

## Refactorización

Se extrajeron los siguientes métodos:

### calcularTotal()

```java
private double calcularTotal(
        LineaPedido[] lineas) {

    return Arrays.stream(lineas)
            .mapToDouble(l ->
                l.getPrecioUnitario()
                * l.getCantidad())
            .sum();
}
```

### aplicarDescuento()

```java
private double aplicarDescuento(
        double total,
        CodigoDescuento descuento) {

    return descuento != null
            ? total * (1 - descuento.getPorcentaje())
            : total;
}
```

### persistirPedido()

```java
private String persistirPedido(
        DatosCliente cliente,
        double total) {

    Pedido pedido =
            new Pedido(
                    cliente.getNombre(),
                    total);

    return "OK_" +
            repo.save(pedido).getId();
}
```

## Beneficios

* Menor complejidad ciclomática.
* Métodos reutilizables.
* Mejor comprensión del flujo de negocio.

---

# Técnica 3 — Extract Class

## Problema

La lógica de notificaciones no pertenecía al dominio de pedidos.

### Código Original

```java
System.out.println(
    "Enviando email a: "
    + clienteEmail);

System.out.println(
    "Pedido urgente: "
    + esUrgente);
```

## Solución

Se creó la clase independiente:

```java
@Service
public class NotificacionService {

    public void notificarPedido(
            DatosCliente cliente,
            boolean urgente) {

        System.out.println(
            "Email enviado a "
            + cliente.getEmail());

    }
}
```

### Inyección por Constructor

```java
@Service
public class PedidoService {

    private final PedidoRepository repo;
    private final NotificacionService notificacion;

    public PedidoService(
            PedidoRepository repo,
            NotificacionService notificacion) {

        this.repo = repo;
        this.notificacion = notificacion;
    }
}
```

## Beneficios

* Separación de responsabilidades.
* Mejor mantenibilidad.
* Facilita pruebas unitarias.

---

# Resultado Final del Método Principal

Después de la refactorización:

```java
public String procesarPedido(
        DatosCliente cliente,
        LineaPedido[] lineas,
        String metodoPago,
        boolean urgente,
        CodigoDescuento descuento) {

    double total = calcularTotal(lineas);

    double totalFinal =
            aplicarDescuento(
                    total,
                    descuento);

    notificacion.notificarPedido(
            cliente,
            urgente);

    return persistirPedido(
            cliente,
            totalFinal);
}
```

Total de líneas: **7**

---

# Métricas SonarQube — Después de la Refactorización

| Métrica                 | Antes  | Después |
| ----------------------- | ------ | ------- |
| Complejidad Ciclomática | 16     | 2       |
| Code Smells             | 18     | 4       |
| Bugs                    | 0      | 0       |
| Vulnerabilidades        | 0      | 0       |
| Technical Debt Ratio    | 6.8%   | 1.2%    |
| Deuda Técnica           | 2h 15m | 18m     |
| Maintainability Rating  | C      | A       |

---

# Comparación de Mejoras

| Indicador               | Mejora |
| ----------------------- | ------ |
| Complejidad Ciclomática | -87.5% |
| Code Smells             | -77.8% |
| Deuda Técnica           | -86.7% |
| Maintainability Rating  | C → A  |

---

# Evidencia SonarQube

## Dashboard Inicial

![Dashboard Inicial](docs/sonar-before.png)

---

## Dashboard Final

![Dashboard Final](docs/sonar-after.png)

---

## Comparación de Métricas

![Comparación](docs/comparison.png)

---

# Verificación de Requisitos

| Requisito                             | Estado |
| ------------------------------------- | ------ |
| Proyecto compila correctamente        | ✅      |
| DatosCliente inmutable                | ✅      |
| Campos final sin setters              | ✅      |
| Extract Method aplicado               | ✅      |
| Extract Class aplicado                | ✅      |
| Value Objects implementados           | ✅      |
| Inyección por constructor             | ✅      |
| Menos Code Smells en SonarQube        | ✅      |
| Comparación antes/después documentada | ✅      |

---

# Historial de Commits

### Commit 1

```text
feat: implementación inicial con code smells intencionales
```

### Commit 2

```text
refactor: aplicar value objects y extract method
```

### Commit 3

```text
refactor: extraer notificaciones a servicio independiente
```

### Commit 4

```text
docs: documentar métricas SonarQube y análisis final
```

---

# Conclusiones

La aplicación de las técnicas Extract Method, Extract Class e Introducción de Value Objects permitió eliminar los principales smells de tipo Bloater presentes en el sistema. Como resultado, la complejidad ciclomática disminuyó de 16 a 2, la deuda técnica se redujo significativamente y el número total de Code Smells pasó de 18 a 4. El análisis realizado con SonarQube confirmó una mejora sustancial en la mantenibilidad del proyecto, elevando el Maintainability Rating desde C hasta A y facilitando futuras tareas de evolución y mantenimiento del software.
