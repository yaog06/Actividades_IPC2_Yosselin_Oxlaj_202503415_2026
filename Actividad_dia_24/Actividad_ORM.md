# Actividad Corta de Laboratorio: De ADO.NET Tradicional a la Automatización con EF Core

**Nombre:** Yosselin Aracely Oxlaj González
**Carné:** 202503415
link del repo: https://github.com/yaog06/Actividades_IPC2_Yosselin_Oxlaj_202503415_2026

---
## Parte 1: Diagnóstico Técnico y Brecha de Impedancia

### 1. La Brecha de Impedancia

La "brecha de impedancia" se refiere a la diferencia conceptual entre cómo se organiza la información en el mundo orientado a objetos (C#) y cómo se organiza en el mundo relacional (SQL). El ORM (Entity Framework Core) actúa como traductor entre ambos dominios, según las siguientes equivalencias:

- Clase Clásica (POCO)  Mapea a ->  **Tabla**                
- Propiedad/Atributo  Mapea a ->    **Columna**              
- Instancia de Objeto  Mapea a ->   **Fila / Registro**      

En resumen: una clase POCO (Plain Old CLR Object) representa la estructura de una tabla, cada propiedad de esa clase corresponde a una columna, y cada objeto instanciado de esa clase representa una fila concreta almacenada en la base de datos.

---

### 2. Mitigación de Vulnerabilidades (Inyección SQL)

- Propiedad nativa de EF Core: Entity Framework Core utiliza de forma parametrizada y nativa la abstracción de consultas LINQ. Al traducir expresiones de C# a SQL Server, delega la inyección de los valores tratándolos exclusivamente como valores literales invariables (parámetros), anulando cualquier compilación de código malicioso.

- Comando equivalente en ADO.NET tradicional: Para mitigar este riesgo en bajo nivel, se utilizaba la clase SqlParameter vinculada explícitamente a la colección Parameters del objeto SqlCommand (por ejemplo, cmd.Parameters.Add()), evitando así la concatenación directa de strings.

---

### 3. Optimización de Infraestructura: `.AsNoTracking()`

El método .AsNoTracking() en las consultas de EF Core desactiva el mecanismo interno de seguimiento de cambios (Change Tracker) del contexto sobre las entidades recuperadas. Dado que las listas o reportes exclusivos de lectura no requieren ser editados ni guardados de vuelta en la base de datos, apagar esta característica libera de forma inmediata memoria RAM crítica en el servidor web de la universidad al no almacenar copias de estado duplicadas por cada fila. Es una muestra de solidaridad computacional ya que optimiza el rendimiento del hardware compartido, previniendo la degradación del servicio para el resto de los estudiantes que concurren al sistema.

---

## Parte 2: Desafío de Refactorización de Código

### Paso 2.1 — El Contexto (DbContext)

```csharp
using Microsoft.EntityFrameworkCore;
 
public class UnidadAcademicaContext : DbContext
{
    // DbSet que mapea la entidad Catedratico a la tabla Tbl_Catedraticos
    public DbSet<Catedratico> Catedraticos { get; set; }
 
    public UnidadAcademicaContext(DbContextOptions<UnidadAcademicaContext> options)
        : base(options)
    {
    }
 
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Se indica explícitamente el nombre real de la tabla en la base de datos
        modelBuilder.Entity<Catedratico>().ToTable("Tbl_Catedraticos");
    }
}
```

### Paso 2.2 — La Consulta LINQ (versión moderna)

```csharp
public List<Catedratico> ObtenerCatedraticosModernos(UnidadAcademicaContext _context)
{
    return _context.Catedraticos
        .AsNoTracking() // Requisito de rendimiento: se apaga el rastreador de cambios
        .Where(c => c.Nombre.StartsWith("Ing."))
        .ToList();
}
```

**Comparación con el código viejo:** mientras que en ADO.NET había que abrir manualmente la conexión, crear el `SqlCommand`, parametrizar el filtro y leer fila por fila con `SqlDataReader` para mapear cada columna a una propiedad, en EF Core toda esa lógica de bajo nivel queda abstraída: la conexión, la traducción a SQL parametrizado y el mapeo objeto-relacional los maneja el ORM automáticamente, y el desarrollador solo expresa la intención de la consulta usando LINQ.

---

## Parte 3: Referencias Bibliográficas

- Facultad de Ingeniería, USAC. (2026). *Sesión 17: Conectividad con SQL Server. Acceso Estructurado a Datos mediante C# y ADO.NET.* Laboratorio de Introducción a la Programación y Computación 2. Guatemala.
- Facultad de Ingeniería, USAC. (2026). *Sesión 18: Mapeo de Objetos Relacionales. Persistencia Automatizada con Entity Framework Core.* Laboratorio de Introducción a la Programación y Computación 2. Guatemala.