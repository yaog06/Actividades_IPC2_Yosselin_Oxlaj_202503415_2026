# Actividad de Laboratorio: Interoperabilidad y Carga Masiva de Datos

**Fecha:** 26 de junio de 2026

Link del repositorio: https://github.com/yaog06/Actividades_IPC2_Yosselin_Oxlaj_202503415_2026.git

Yosselin Aracely Oxlaj González
202503415

---

## Parte 1: Evaluación Conceptual y Buenas Prácticas

### Formatos de Intercambio: Completa la siguiente tabla comparativa de formatos masivos según las ventajas y desventjas exúestas en la sesión:

| Formato | Ventajas | Desventajas |
| :--- | :--- | :--- |
| **CSV** | Extremadamente ligero y fácil de generar desde herramientas como Excel, ya que almacena datos tabulados en filas y columnas. | No soporta jerarquías complejas, limitándose únicamente a datos planos (un ejemplo seria llevar el control de productos de una tienda, la cual se guardaria su informacion importante, es decir, un inventario). |
| **XML** | Estructurado, soporta tipos de datos explícitos y jerarquías organizadas. | Es verboso, lo que genera archivos significativamente más pesados que JSON o CSV. |

### 1. Diferencia De Procesos: 
Utilizando la librería nativa `System.Text.Json`, la diferencia técnica radica en la dirección del flujo de la transformación:
*   **Serialización:** Es el proceso de convertir objetos C# a formato JSON, útil para enviar información a través de la red.
*   **Deserialización:** Es el proceso inverso, donde una cadena de texto en formato JSON se procesa y transforma nuevamente en un objeto estructurado en C# para poder manipular sus propiedades internamente.

### 2. El Antipatrón del Rendimiento:
El error de rendimiento **N+1** en la lectura de archivos masivos consiste en realizar una operación individual en la base de datos o una petición HTTP externa por cada una de las filas (o registros) que contiene el archivo. Si el documento posee miles de filas, este comportamiento destruye el rendimiento del sistema debido a la latencia acumulada por cada viaje al servidor.

**Estrategia de optimización (Batching):**  
Para solucionarlo, se debe aplicar el procesamiento por lotes, el cual consiste en leer secuencialmente el archivo, acumular los registros estructurados en una colección en memoria intermedia (como una lista) y persistirlos todos juntos mediante un único comando SQL optimizado o una sola transacción en la base de datos.

---

## Parte 2: Implementación práctica e C#

### Desafío 1: Consumo de Endpoints y Deserialización

```csharp
using System;
using System.Net.Http;
using System.Text.Json;
using System.Threading.Tasks;

public class AlumnoService
{
    // Instancia única de HttpClient para evitar el agotamiento de sockets
    private static readonly HttpClient client = new HttpClient();

    public async Task<Alumno> ObtenerAlumnoAsync()
    {
        string url = "[https://api.usac.edu/v1/alumnos](https://api.usac.edu/v1/alumnos)";

        try
        {
            // Realizar la petición GET de forma asíncrona
            HttpResponseMessage response = await client.GetAsync(url);
            
            // Valida el código de estado y lanza HttpRequestException si no es 2xx
            response.EnsureSuccessStatusCode();

            // Leer el cuerpo de la respuesta como una cadena de texto
            string json = await response.Content.ReadAsStringAsync();

            // Configurar las opciones para ignorar mayúsculas/minúsculas en las propiedades
            var options = new JsonSerializerOptions
            {
                PropertyNameCaseInsensitive = true
            };

            // Deserializar el payload JSON al objeto Alumno
            Alumno alumno = JsonSerializer.Deserialize<Alumno>(json, options);
            return alumno;
        }
        catch (HttpRequestException ex)
        {
            // Error de red o código de error HTTP inválido
            Console.WriteLine($"Error de red al consumir la API: {ex.Message}");
            throw;
        }
        catch (JsonException ex)
        {
            // El formato JSON recibido no coincide con la clase objetivo
            Console.WriteLine($"Error de deserialización JSON: {ex.Message}");
            throw;
        }
    }
}
```
### Desafío 2: Endpoint para Carga Masiva CSV

- Uso de StreamReader y ReadLineAsync(): En lugar de cargar todo el archivo de golpe en la memoria RAM (lo cual causaría una caída del servidor con archivos muy grandes), el código abre un flujo de datos (OpenReadStream()) y lee el CSV línea por línea de forma asíncrona.

- Estrategia de Batching (Evitar el problema N+1): En lugar de hacer un _context.Estudiantes.Add(estudiante) y un SaveChangesAsync() dentro del ciclo por cada fila del archivo (lo que causaría el antipatrón N+1), el código va guardando los objetos en una lista intermedia en memoria (listaEstudiantes.Add(...)).

- Inserción única al final: Una vez que el ciclo termina de leer todo el archivo, se utiliza _context.Estudiantes.AddRange(listaEstudiantes) y un único await _context.SaveChangesAsync(). Esto envía todos los registros juntos a la base de datos en una sola transacción eficiente.
```csharp
using System.Collections.Generic;
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class ControlAcademicoController : ControllerBase
{
    private readonly ControlAcademicoContext _context;

    public ControlAcademicoController(ControlAcademicoContext context)
    {
        _context = context;
    }

    [HttpPost("upload-csv")]
    public async Task<IActionResult> Upload(IFormFile file)
    {
        // Validación de seguridad básica del archivo
        if (file == null || file.Length == 0)
        {
            return BadRequest("Por favor, proporcione un archivo CSV válido.");
        }

        var listaEstudiantes = new List<Estudiante>();

        // Uso de Streams para procesar el archivo sin saturar la memoria RAM
        using (var stream = file.OpenReadStream())
        using (var reader = new StreamReader(stream))
        {
            while (!reader.EndOfStream)
            {
                // Lectura asíncrona línea por línea
                string line = await reader.ReadLineAsync();
                
                if (string.IsNullOrWhiteSpace(line)) continue;

                var values = line.Split(',');

                // Mapeo manual elemental (Asumiendo Posición 0: Carné, Posición 1: Nombre)
                var estudiante = new Estudiante
                {
                    Carne = values[0].Trim(),
                    Nombre = values[1].Trim()
                };

                listaEstudiantes.Add(estudiante);
            }
        }

        // Carga por lotes (Batching) eficiente optimizada por Entity Framework Core
        _context.Estudiantes.AddRange(listaEstudiantes);
        await _context.SaveChangesAsync();

        return Ok(new { TotalProcesados = listaEstudiantes.Count, Mensaje = "Carga masiva realizada con éxito." });
    }
}
```

## Parte 3: Referencias Bibliográficas
Facultad de Ingeniería, USAC. (2026). Sesión 20: Integración de Datos. Consumo de APIs
Externas y Carga Masiva (CSV/XML). Laboratorio del curso Introducción a la
Programación y Computación 2. Guatemala.
