# PracticaB
# Repaso completo — Estructuras del parcial (API en capas .NET)

Cada bloque de código tiene, debajo, el comentario de qué hace y por qué. Usalo para repasar antes del examen, no como copia-pega ciega: entendé cada línea.

---

## 1. Config tipada — clase que espeja una sección del `appsettings.json`

```csharp
namespace Copa.Data.Config;

public sealed class CopaConfig
{
    public string Titulo { get; set; } = string.Empty;
    public string ArchivoDatos { get; set; } = string.Empty;
    public int JugadoresPorPagina { get; set; }
    public CredencialesConfig Credenciales { get; set; } = new();
}

public sealed class CredencialesConfig
{
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
}
```

**Qué hace:** define una clase en C# con una propiedad por cada clave del JSON, mismo nombre, mismo "nivel". `Credenciales` es un objeto anidado en el JSON → por eso es otra clase (`CredencialesConfig`), no una propiedad suelta. Los `string` llevan `= string.Empty` para no arrancar en `null` (evita `NullReferenceException` y calla warnings de nullable). Los `int` no necesitan inicializador, arrancan en 0 solos. `Credenciales` lleva `= new()` porque es un objeto, no puede quedar en null.

---

## 2. Bind de la configuración en `Program.cs`

```csharp
builder.Services
    .AddOptions<CopaConfig>()
    .Bind(builder.Configuration.GetSection("Copa"))
    .ValidateOnStart();
```

**Qué hace:** `AddOptions<CopaConfig>()` registra la clase en el contenedor de dependencias (para poder inyectarla después). `.Bind(GetSection("Copa"))` es el "casamiento" real: toma la sección `Copa` del `appsettings.json` y rellena una instancia de `CopaConfig` con esos valores. El nombre entre comillas tiene que ser **exactamente** igual a la clave del JSON. `.ValidateOnStart()` hace que la app falle al arrancar si algo de la config no es válido, en vez de fallar más tarde en medio de una petición.

---

## 3. JwtConfig — config de seguridad (mismo patrón, sin anidados)

```csharp
namespace Copa.Api.Security;

public sealed class JwtConfig
{
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public string SecretKey { get; set; } = string.Empty;
    public int ExpiresMinutes { get; set; }
}
```

**Qué hace:** igual idea que `CopaConfig`, pero para la sección `Jwt`. `Issuer`/`Audience` identifican quién emite y para quién es el token. `SecretKey` es la clave con la que se firma y valida el token (nadie más la conoce, por eso nadie puede falsificar un token válido). `ExpiresMinutes` define cuánto dura el token antes de vencer.

⚠️ El bind de `JwtConfig` en `Program.cs` suele aparecer **dos veces**: una con `AddOptions` (para inyectarlo en endpoints), y otra leyéndolo directo con `.Get<JwtConfig>()` (porque se necesita ya mismo, para configurar la validación de tokens antes de que la app termine de armarse). Revisar que **las dos** apunten a la sección correcta.

---

## 4. DTO genérico de paginación

```csharp
namespace Copa.Business.Dtos;

public sealed record PaginaResult<T>(
    int Page,
    int PageSize,
    int TotalItems,
    int TotalPages,
    IReadOnlyList<T> Items);
```

**Qué hace:** `record` es una forma corta de declarar una clase que solo transporta datos: el compilador arma solo el constructor y las propiedades. El `<T>` lo hace **genérico**: sirve para paginar cualquier tipo de dato (jugadores, partidos, lo que sea), no está atado a uno solo. Cuando se usa como `PaginaResult<JugadorDto>`, ese `T` se convierte en `JugadorDto`. Las propiedades van con mayúscula inicial (convención de C#); al serializar a JSON, .NET las baja a minúscula automáticamente (`Page` → `"page"`).

---

## 5. Servicio — método de paginado completo

```csharp
public async Task<PaginaResult<JugadorDto>> GetPaginadoAsync(int page, CancellationToken cancellationToken = default)
{
    if (page < 1)
    {
        throw new ArgumentOutOfRangeException(nameof(page), "La pagina debe ser mayor o igual a 1.");
    }

    var jugadores = await repository.GetAllAsync(cancellationToken);

    var pageSize = _config.JugadoresPorPagina;
    var totalItems = jugadores.Count();
    var totalPages = totalItems / pageSize;
    var items = jugadores
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(ToDto)
        .ToList();

    return new PaginaResult<JugadorDto>(page, pageSize, totalItems, totalPages, items);
}
```

**Qué hace, línea por línea:**
- `if (page < 1)` → valida el parámetro antes de usarlo; si viene mal, corta con una excepción (el endpoint la traduce a HTTP 400).
- `repository.GetAllAsync(...)` → trae **todos** los jugadores desde la capa Data (que lee el JSON). `jugadores` es una lista completa.
- `pageSize = _config.JugadoresPorPagina` → cuántos van por página, sacado de la configuración (no hardcodeado).
- `totalItems = jugadores.Count()` → cuántos hay en total.
- `totalPages = totalItems / pageSize` → cuántas páginas salen de dividir el total por el tamaño de página.
- `Skip((page - 1) * pageSize)` → saltea los elementos de las páginas anteriores a la pedida.
- `Take(pageSize)` → agarra solo los que corresponden a esta página.
- `Select(ToDto)` → convierte cada `Jugador` (entidad interna) en `JugadorDto` (lo que se expone hacia afuera).
- `ToList()` → materializa el resultado en una lista concreta.
- `return new PaginaResult<JugadorDto>(page, pageSize, totalItems, totalPages, items)` → arma el objeto de respuesta. **El orden de los argumentos tiene que coincidir con el orden de los parámetros del record** (se conectan por posición, no por nombre).

---

## 6. DTO de resultado de búsqueda

```csharp
namespace Copa.Business.Dtos;

public sealed record BusquedaResult(
    string Texto,
    int TotalItems,
    IReadOnlyList<JugadorDto> Items);
```

**Qué hace:** guarda el texto que se buscó, cuántos resultados matchearon, y la lista de esos resultados ya convertidos a DTO. No es genérico (`<T>`) porque este resultado siempre es de jugadores, a diferencia del `PaginaResult` que se pensó reutilizable.

---

## 7. Servicio — método de búsqueda con filtro WHERE OR

```csharp
public async Task<BusquedaResult> BuscarAsync(string texto, CancellationToken cancellationToken = default)
{
    if (string.IsNullOrWhiteSpace(texto))
    {
        throw new ArgumentException("El texto no puede estar vacio.", nameof(texto));
    }

    var jugadores = await repository.GetAllAsync(cancellationToken);

    var jugadoresFiltrados = jugadores
        .Where(j => j.NombreCompleto.Contains(texto, StringComparison.OrdinalIgnoreCase) ||
                    j.Seleccion.Contains(texto, StringComparison.OrdinalIgnoreCase))
        .Select(ToDto)
        .ToList();

    return new BusquedaResult(texto, jugadoresFiltrados.Count, jugadoresFiltrados);
}
```

**Qué hace:**
- `string.IsNullOrWhiteSpace(texto)` → valida que no venga vacío ni solo espacios.
- `jugadores.Where(j => ...)` → filtra la lista completa, quedándose solo con los elementos que cumplen la condición. `j` representa **un** jugador en cada vuelta del filtro.
- La condición usa `||` (OR): el jugador entra si **su nombre contiene el texto O su selección lo contiene**. Con `&&` (AND) tendría que cumplir las dos a la vez, que no es lo pedido.
- `.Contains(texto, StringComparison.OrdinalIgnoreCase)` → busca el texto como substring, ignorando mayúsculas/minúsculas. Distinto de `.Equals(...)`, que exige coincidencia exacta completa.
- `Select(ToDto)` + `ToList()` → igual que antes, convierte y materializa.
- El `return` respeta el orden del record: `Texto`, `TotalItems` (acá `jugadoresFiltrados.Count`, la cuenta de la lista ya filtrada), `Items` (la lista en sí).

---

## 8. Método auxiliar de mapeo (entidad → DTO)

```csharp
private static JugadorDto ToDto(Jugador jugador)
{
    return new JugadorDto(
        jugador.CodigoJugador,
        jugador.NombreCompleto,
        jugador.Seleccion,
        jugador.SeleccionSigla,
        jugador.Puesto,
        jugador.Dorsal,
        jugador.Tantos,
        jugador.Encuentros,
        jugador.Edad);
}
```

**Qué hace:** convierte **un** objeto `Jugador` (modelo interno, capa Data) en **un** `JugadorDto` (lo que se expone en la API). Es `private` porque solo lo usa esta clase, y `static` porque no necesita nada del estado de la instancia (`_config`, etc.), solo el parámetro que recibe. Se usa dentro de `.Select(ToDto)` para transformar listas enteras, uno por uno.

---

## 9. Endpoint de login/token (no se suele tocar, pero hay que saber leerlo)

```csharp
app.MapPost("/api/auth/token", (
    [FromBody] TokenRequest request,
    IOptions<CopaConfig> copaOptions,
    IOptions<JwtConfig> jwtOptions) =>
{
    var credentials = copaOptions.Value.Credenciales;
    if (!string.Equals(request.Username, credentials.Username, StringComparison.Ordinal) ||
        !string.Equals(request.Password, credentials.Password, StringComparison.Ordinal))
    {
        return Results.Unauthorized();
    }

    var jwt = jwtOptions.Value;
    var expires = DateTime.UtcNow.AddMinutes(jwt.ExpiresMinutes);
    var claims = new[]
    {
        new Claim(JwtRegisteredClaimNames.Sub, request.Username),
        new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString("N"))
    };

    var token = new JwtSecurityToken(
        issuer: jwt.Issuer,
        audience: jwt.Audience,
        claims: claims,
        expires: expires,
        signingCredentials: new SigningCredentials(
            new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwt.SecretKey)),
            SecurityAlgorithms.HmacSha256));

    var accessToken = new JwtSecurityTokenHandler().WriteToken(token);
    return Results.Ok(new TokenResponse(accessToken, "Bearer", jwt.ExpiresMinutes * 60));
})
.AllowAnonymous()
.WithName("CreateToken");
```

**Qué hace:**
- `[FromBody] TokenRequest request` → toma el JSON del cuerpo del POST (`{ "username": "...", "password": "..." }`) y lo mete en un objeto.
- `IOptions<CopaConfig> copaOptions` / `IOptions<JwtConfig> jwtOptions` → pide las dos configs por inyección de dependencias.
- Compara `request.Username/Password` contra `credentials.Username/Password` (que vienen de la config, **nunca hardcodeados**). Si no coinciden → `Results.Unauthorized()` (HTTP 401).
- Si coinciden: arma el `expires` (ahora + minutos de duración), los `claims` (datos dentro del token: quién es, un id único), y firma todo con `SecretKey` usando `HmacSha256`.
- `WriteToken(token)` → convierte el token armado en el string largo (`eyJhbGc...`) que se envía al cliente.
- Devuelve `Results.Ok(...)` con `accessToken`, `tokenType` ("Bearer") y `expiresIn` (en segundos, por eso `* 60`).
- `.AllowAnonymous()` → este endpoint **no** pide token para acceder (obvio: es el que entrega el token).

---

## 10. Endpoints protegidos (paginado y búsqueda)

```csharp
app.MapGet("/api/jugador", async (
    [FromQuery] int? page,
    IJugadorService jugadorService,
    CancellationToken cancellationToken) =>
{
    var currentPage = page ?? 1;
    if (currentPage < 1)
    {
        return Results.BadRequest(new { error = "La pagina debe ser mayor o igual a 1." });
    }

    var result = await jugadorService.GetPaginadoAsync(currentPage, cancellationToken);
    return Results.Ok(result);
})
.RequireAuthorization()
.WithName("GetJugadores");


app.MapGet("/api/jugador/buscar/{texto}", async (
    string texto,
    IJugadorService jugadorService,
    CancellationToken cancellationToken) =>
{
    if (string.IsNullOrWhiteSpace(texto))
    {
        return Results.BadRequest(new { error = "El texto es requerido." });
    }

    var result = await jugadorService.BuscarAsync(texto, cancellationToken);
    return Results.Ok(result);
})
.RequireAuthorization()
.WithName("GetJugadoresPorTexto");
```

**Qué hace:**
- `MapGet` → registra el endpoint para el verbo HTTP GET (si la consigna pide GET y el código dice `MapPost`, el endpoint no se encuentra al pedirlo por GET → 404).
- `[FromQuery] int? page` → toma `page` de la query string (`?page=2`); es `int?` (nullable) porque puede no venir, y ahí se usa `page ?? 1` (si es null, default a 1).
- `string texto` en la ruta (`{texto}`) → se toma directo del segmento de la URL.
- Cada handler valida la entrada, llama al método del **service** (nunca hace la lógica de negocio ahí mismo), y devuelve `Results.Ok(...)` o `Results.BadRequest(...)`.
- `.RequireAuthorization()` → exige un JWT válido en el header `Authorization: Bearer ...`. Sin token válido, responde 401 automáticamente, antes de ejecutar el código del handler.

---

## 11. Repositorio (capa Data) — lectura del archivo, ya suele venir hecho

```csharp
public sealed class JugadorRepository(IOptions<CopaConfig> options) : IJugadorRepository
{
    private readonly CopaConfig _config = options.Value;

    public async Task<IReadOnlyList<Jugador>> GetAllAsync(CancellationToken cancellationToken = default)
    {
        var path = ResolveJsonPath(_config.ArchivoDatos);
        await using var stream = File.OpenRead(path);
        var document = await JsonSerializer.DeserializeAsync<CopaJsonDocument>(stream, JsonOptions, cancellationToken);

        return document?.Jugadores ?? [];
    }
}
```

**Qué hace:** lee el nombre del archivo desde la config (`_config.ArchivoDatos`), abre ese archivo, lo deserializa (convierte el JSON crudo en objetos C#) usando el molde `CopaJsonDocument`, y devuelve la lista de jugadores que trae adentro. Si por algún motivo el documento es null, devuelve una lista vacía (`?? []`) en vez de romper.

---

## 12. Tabla resumen — quién hace qué en cada capa

| Capa | Responsabilidad | Ejemplos |
|---|---|---|
| **Data** | Leer/escribir datos crudos. No sabe de HTTP ni de reglas de negocio. | `JugadorRepository`, `Jugador` (modelo), `CopaConfig` |
| **Business** | Reglas de negocio: paginar, filtrar, transformar a DTO. No sabe de HTTP. | `JugadorService`, `JugadorDto`, `PaginaResult`, `BusquedaResult` |
| **Api** | Exponer HTTP: recibe requests, valida entrada superficial, llama al service, devuelve respuestas. | `Program.cs` (endpoints), `JwtConfig` |
| **Tests** | Verificar que todo funcione junto, de punta a punta. | Los `[Fact]` que ya vienen escritos |

---

## 13. Glosario rápido de palabras clave

| Palabra | Qué significa |
|---|---|
| `record` | Tipo de clase corto para transportar datos, con constructor y propiedades auto-generados |
| `sealed` | Nadie puede heredar de esta clase (convención de orden, no obligatorio) |
| `IOptions<T>` | Forma de recibir una config tipada por inyección de dependencias |
| `.Value` | Sobre un `IOptions<T>`, te da la instancia real ya rellenada |
| `Skip(n)` | Salta los primeros `n` elementos de una secuencia |
| `Take(n)` | Toma los primeros `n` elementos de lo que queda |
| `Where(condición)` | Filtra: deja solo los elementos que cumplen la condición |
| `Select(función)` | Transforma cada elemento aplicando la función |
| `.Contains(x, StringComparison...)` | ¿El texto contiene `x` como substring? |
| `.Equals(x, StringComparison...)` | ¿El texto es exactamente igual a `x`? |
| `OrdinalIgnoreCase` | Comparación de texto ignorando mayúsculas/minúsculas |
| `RequireAuthorization()` | El endpoint exige JWT válido |
| `AllowAnonymous()` | El endpoint no exige JWT |
| `[FromBody]` | El dato viene del cuerpo (JSON) del request |
| `[FromQuery]` | El dato viene de la query string (`?clave=valor`) |
| `CancellationToken` | Permite cancelar una operación async si el cliente corta la conexión |
