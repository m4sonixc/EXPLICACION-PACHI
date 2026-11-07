Aquí tienes el **Manual de Defensa Técnica del Código Fuente de PachiTaller**. Está redactado paso a paso, archivo por archivo, explicando **qué hace cada línea importante, qué significan las palabras técnicas de C# y XAML, y cómo responder si el profesor te pregunta por cualquier parte del proyecto**. *(Cuando una estructura ya fue explicada en una pantalla anterior, se indica para no repetir lo mismo).*

---

# MANUAL DE EXPLICACIÓN Y DEFENSA DEL CÓDIGO FUENTE: PACHITALLER

---

## CAPÍTULO 1: Archivos de Arranque y Configuración Global

### 1.1. `Constants.cs` (Credenciales de Conexión)
```csharp
public static class Constants
{
    public const string SupabaseUrl = "https://tu-proyecto.supabase.co";
    public const string SupabaseAnonKey = "eyJhbGciOi...";
}
```
* **¿Qué hace este archivo?** Guarda en un solo lugar la dirección de internet (`SupabaseUrl`) y la llave pública de permiso (`SupabaseAnonKey`) de nuestra base de datos en Supabase.
* **¿Qué significa `static` y `const`?**
  * `static`: Significa que no hace falta crear un objeto con `new Constants()` para usarla; se puede llamar desde cualquier parte poniendo `Constants.SupabaseUrl`.
  * `const` (Constante): Significa que ese valor es fijo y nunca cambia mientras la aplicación está funcionando.
* **Detalle técnico de defensa:** La URL no debe terminar con barra `/` porque el paquete de Supabase le agrega automáticamente `/rest/v1/` al final. Si tuviera doble barra, el servidor daría error `Unsupported method`.

---

### 1.2. `MauiProgram.cs` (El Motor de Arranque e Inyección de Dependencias)
```csharp
builder.Services.AddSingleton<SupabaseService>();
builder.Services.AddTransient<LoginPage>();
// ... resto de las páginas
```
* **¿Qué hace este archivo?** Es el primer archivo que se ejecuta cuando abres la app en el celular. Configura las fuentes de letra y registra los servicios y pantallas en el **Contenedor de Inyección de Dependencias**.
* **Pregunta clave de examen: ¿Cuál es la diferencia entre `AddSingleton` y `AddTransient`?**
  * **`AddSingleton<SupabaseService>()`:** Crea **una única instancia** de la conexión a Supabase para toda la vida de la aplicación. Así no abrimos 20 conexiones distintas a internet cada vez que cambiamos de pantalla; todas las pantallas comparten el mismo puente hacia la base de datos.
  * **`AddTransient<LoginPage>()`:** Crea **una ventana nueva y limpia** cada vez que navegamos hacia esa pantalla, y libera la memoria cuando salimos de ella.

---

### 1.3. `App.xaml` y `App.xaml.cs` (Recursos Globales y Pantalla Inicial)
En `App.xaml`:
```xml
<converters:EstadoColorConverter x:Key="EstadoColorConverter" />
```
En `App.xaml.cs`:
```csharp
public App()
{
    InitializeComponent();
    MainPage = new AppShell();
}
```
* **¿Qué hacen?**
  * `App.xaml` declara recursos que estarán disponibles en **toda** la aplicación, como los estilos de colores y nuestro `EstadoColorConverter` (que sirve para pintar de colores los estados de los turnos).
  * `App.xaml.cs` le dice a la aplicación: *"Apenas arranques, tu pantalla principal (`MainPage`) será `AppShell` (que muestra el Login)"*.

---

## CAPÍTULO 2: Capa de Servicios (`Services/`)

### 2.1. `Services/SupabaseService.cs` (Puente con la Base de Datos en la Nube)
```csharp
public class SupabaseService
{
    public Client Client { get; private set; }

    public SupabaseService()
    {
        var options = new SupabaseOptions
        {
            AutoRefreshToken = true,
            AutoConnectRealtime = true
        };
        Client = new Client(Constants.SupabaseUrl, Constants.SupabaseAnonKey, options);
    }

    public async Task InitializeAsync()
    {
        await Client.InitializeAsync();
    }
}
```
* **¿Qué hace?** Envuelve al cliente oficial del paquete NuGet `supabase-csharp`.
* **¿Qué significan sus partes?**
  * `Client`: Es el objeto que nos permite hacer consultas como `.From<Turno>().Get()`, `.Insert()`, `.Update()` y `.Delete()`.
  * `async Task` y `await`: Indican que la conexión por internet es **asíncrona**. Es decir, mientras el celular se conecta con el servidor de Supabase, la pantalla del teléfono no se congela.

---

### 2.2. `Services/SessionService.cs` (Memoria de quién inició sesión)
```csharp
public static class SessionService
{
    public static Usuario? CurrentUser { get; set; }

    public static void Logout()
    {
        CurrentUser = null;
    }
}
```
* **¿Qué hace?** Guarda en la memoria RAM del teléfono los datos de la persona que acaba de iniciar sesión (`CurrentUser`).
* **¿Para qué nos sirve en el resto de la app?**
  1. Para mostrar arriba *"Bienvenido, Marcos Cabrera"*.
  2. Para que cuando el cliente pida un turno, la app ya sepa automáticamente su `Id`, su `Nombre` y su `Telefono` sin que tenga que volver a escribirlos.
  3. Cuando toca el botón **Salir**, llama a `Logout()`, que vacía `CurrentUser = null` y lo devuelve al Login.

---

### 2.3. `Services/PdfService.cs` (Generador de Comprobantes y Tickets)
```csharp
var nombreArchivo = $"Comprobante_{turno.Patente}_{DateTime.Now:yyyyMMdd_HHmm}.html";
var rutaCompleta = Path.Combine(FileSystem.CacheDirectory, nombreArchivo);
// ... arma la estructura HTML con CSS ...
File.WriteAllText(rutaCompleta, htmlContent);
return rutaCompleta;
```
* **¿Qué hace?** Arma un documento con diseño de factura formal (con encabezado de PachiTaller, recuadro de cliente y vehículo, tabla de repuestos utilizados, mano de obra y total) y lo guarda como archivo dentro del teléfono para poder compartirlo por WhatsApp o abrirlo en el navegador.
* **Explicación de sus partes clave:**
  * `FileSystem.CacheDirectory`: Es una carpeta temporal segura que .NET MAUI nos da dentro de Android o Windows donde la app tiene permiso para crear archivos sin pedir permisos complicados al usuario.
  * `Task.Run(() => { ... })`: Ejecuta la creación del archivo en un hilo secundario del procesador para que sea instantáneo.
  * `File.WriteAllText(rutaCompleta, htmlContent)`: Escribe el código de la factura en el archivo físico y devuelve la ruta donde quedó guardado para que luego la pantalla lo comparta con `ShareFileRequest`.
* **¿Por qué lo hicimos con plantilla HTML/CSS en vez de librerías de PDF de escritorio?** Porque en Android no existen las fuentes de letra de Windows (como *Arial*), lo que hacía fallar a las librerías viejas. Con HTML/CSS nativo funciona en el 100% de los teléfonos sin errores.

---

## CAPÍTULO 3: Capa de Modelos (`Models/`)

Los modelos son clases de C# que representan las tablas de nuestra base de datos en Supabase.

### 3.1. `Models/Usuario.cs` y `Models/Repuesto.cs`
```csharp
[Table("usuarios")]
public class Usuario : BaseModel
{
    [PrimaryKey("id", false)]
    public long Id { get; set; }

    [Column("username")]
    public string Username { get; set; } = string.Empty;
    // ...
}
```
* **¿Qué significa cada etiqueta (Atributo) entre corchetes `[...]`?**
  * `: BaseModel`: Indica que la clase hereda del modelo base de PostgREST (Supabase). Sin esto, Supabase no sabría leer ni guardar este objeto.
  * `[Table("usuarios")]`: Le dice a C# con qué tabla exacta de Supabase se conecta esta clase.
  * `[PrimaryKey("id", false)]`: Indica que `id` es la clave primaria. **¿Por qué lleva `false` (`shouldInsert: false`)?** Porque el `id` lo inventa automáticamente la base de datos (1, 2, 3...). Poniendo `false`, le decimos a C# que **no envíe el ID cuando creamos un registro nuevo**, evitando errores al insertar.
  * `[Column("username")]`: Conecta la propiedad `Username` de C# (con mayúscula) con la columna `username` de PostgreSQL (en minúscula).

En `Repuesto.cs` además agregamos una propiedad calculada:
```csharp
[JsonIgnore]
public bool EsStockBajo => Stock <= 3;
```
* **¿Qué hace `EsStockBajo`?** Devuelve `true` (verdadero) automáticamente si al repuesto le quedan 3 o menos unidades. El XAML lee esta propiedad para mostrar u ocultar el cartel rojo de **`⚠️ STOCK CRÍTICO`**.

---

### 3.2. `Models/Turno.cs` y `Models/TurnoRepuesto.cs`
*(Tienen la misma estructura de `[Table]`, `[PrimaryKey]` y `[Column]` explicada en el punto 3.1, por lo que omitimos esa parte y explicamos lo nuevo):*

```csharp
[JsonIgnore]
[System.Text.Json.Serialization.JsonIgnore]
public string NombreCliente { get; set; } = string.Empty;

[JsonIgnore]
[System.Text.Json.Serialization.JsonIgnore]
public string FechaFinalizacionFormateada => 
    (FechaFinalizacion ?? CreatedAt ?? DateTime.Now).ToString("dd/MM/yyyy HH:mm");
```
* **Pregunta fija del profesor: ¿Por qué pusieron `[JsonIgnore]` en `NombreCliente`, `TelefonoCliente` y `NombreRepuesto`?**
  * **Respuesta:** En la base de datos, la tabla `turnos` solo guarda el número `usuario_id` (por ejemplo, `5`), no guarda el texto `"Marcos Cabrera"` para no repetir datos. Pero en la pantalla nosotros sí queremos mostrar `"Cliente: Marcos Cabrera"`. Entonces creamos la propiedad `NombreCliente` en C# para mostrarla en la tarjeta, y le pusimos **`[JsonIgnore]`** para que cuando guardemos el turno en Supabase, **esa propiedad se ignore y no viaje por internet**, evitando el error `Could not find column NombreCliente in schema`.
* **¿Qué hace el operador `??` en `FechaFinalizacionFormateada`?**
  * Se llama *operador de coalescencia nula*. Significa: *"Muestra la `FechaFinalizacion`; si está vacía (`null`), usa `CreatedAt`; y si también está vacía, usa la fecha de hoy"*. Así nos aseguramos de que en el historial **nunca** quede la fecha en blanco.

---

### 3.3. `Models/RepuestoVentaItem.cs` (Cálculo en vivo al vender repuestos)
```csharp
public class RepuestoVentaItem : INotifyPropertyChanged
{
    private int _cantidadVenta;
    public int CantidadVenta
    {
        get => _cantidadVenta;
        set
        {
            if (_cantidadVenta != value)
            {
                _cantidadVenta = value;
                OnPropertyChanged();
                OnPropertyChanged(nameof(Subtotal));
            }
        }
    }

    public decimal Subtotal => CantidadVenta * PrecioUnitario;
}
```
* **¿Qué hace esta clase y qué es `INotifyPropertyChanged`?**
  * Cuando el administrador está en la pantalla de **Venta de Repuestos** y escribe un número en la casilla de cantidad (por ejemplo, escribe `2`), la interfaz gráfica necesita enterarse en ese mismo milisegundo de que el `Subtotal` cambió (por ejemplo, de `$0` a `$180.000`).
  * `INotifyPropertyChanged` es una interfaz de .NET que "avisa" a la pantalla cada vez que cambia el valor de `CantidadVenta`, haciendo que el texto `Subtotal: $...` se actualice solo sin tocar ningún botón.

---

## CAPÍTULO 4: Convertidores Visuales (`Converters/`)

### 4.1. `Converters/EstadoColorConverter.cs`
```csharp
public class EstadoColorConverter : IValueConverter
{
    public object Convert(object? value, Type targetType, object? parameter, CultureInfo culture)
    {
        var estado = value?.ToString();
        return estado switch
        {
            "Solicitado" => Colors.DarkOrange,
            "Aceptado" => Colors.SeaGreen,
            "Reprogramado" => Colors.DarkMagenta,
            "En Proceso" => Colors.DodgerBlue,
            "Finalizado" => Colors.ForestGreen,
            "Cancelado" => Colors.Crimson,
            _ => Colors.Gray
        };
    }
}
```
* **¿Qué hace?** Implementa `IValueConverter` (Convertidor de Valores). En la base de datos el estado es un simple texto (`"Finalizado"`), pero la propiedad `TextColor` de un `Label` en XAML necesita un objeto de tipo `Color`. Este convertidor recibe la palabra y devuelve el color correspondiente usando una expresión `switch` moderna de C#.

---

## CAPÍTULO 5: Navegación y Shells (`AppShell`, `AdminShell`, `UserShell`)

En .NET MAUI, **`Shell`** es el contenedor que maneja la navegación entre pantallas y la flecha superior para volver atrás.

En `AdminShell.xaml.cs` y `UserShell.xaml.cs`:
```csharp
Routing.RegisterRoute(nameof(AdminDetalleTurnoPage), typeof(AdminDetalleTurnoPage));
```
* **¿Por qué tenemos 3 Shells distintos (`AppShell`, `AdminShell` y `UserShell`)?**
  * Para separar por completo los ambientes por seguridad y comodidad:
    1. `AppShell`: Solo contiene el Login, Registro y Recuperar Contraseña.
    2. `AdminShell`: Tiene como página raíz el Panel del Administrador y registra las rutas del taller (Detalle de turno, Historial, Catálogo, Venta de repuestos y Balance).
    3. `UserShell`: Tiene como página raíz el Panel del Cliente y registra las rutas de Solicitar Turno e Historial de Vehículos.
* **¿Cómo funciona la navegación en todas las páginas del proyecto?**
  * **Para ir a una pantalla:** `await Shell.Current.GoToAsync(nameof(NombreDePagina));`
  * **Para volver atrás (botón Volver o flecha de arriba):** `await Shell.Current.GoToAsync("..");` (los dos puntos `".."` significan "subir un nivel hacia atrás").

---

## CAPÍTULO 6: Módulo de Autenticación (`Views/LoginPage`, `RegisterPage`, `ForgotPasswordPage`)

### 6.1. Conceptos básicos del diseño XAML (Se repiten en todas las vistas del proyecto)
En todas las pantallas `.xaml` verás estas etiquetas fundamentales:
* `<ContentPage>`: Es una pantalla completa de la aplicación.
* `<ScrollView>`: Permite hacer scroll con el dedo hacia abajo si el contenido no entra en pantallas chicas o cuando se abre el teclado del celular.
* `<VerticalStackLayout>`: Apila los controles uno debajo del otro en forma vertical.
* `<HorizontalStackLayout>`: Coloca los controles uno al lado del otro en fila.
* `<Grid RowDefinitions="..." ColumnDefinitions="...">`: Crea una cuadrícula de filas y columnas. Cuando dice `*` significa *"ocupa todo el espacio libre que sobre"*, y cuando dice `Auto` significa *"ocupa solo lo que mida su contenido"*.
* `<Frame>`: Es una tarjeta con bordes redondeados (`CornerRadius`), color de fondo (`BackgroundColor`) y borde (`BorderColor`).
* `<Entry>`: Es una caja de texto de una sola línea (con `IsPassword="True"` oculta las letras con asteriscos, y `MaxLength="6"` impide escribir más de 6 caracteres).
* *(Nota: En los siguientes capítulos no volveremos a explicar estas etiquetas básicas de XAML ya que funcionan exactamente igual en todas las vistas).*

---

### 6.2. Lógica de `LoginPage.xaml.cs`
```csharp
var response = await _supabaseService.Client
    .From<Usuario>()
    .Filter("username", Postgrest.Constants.Operator.Equals, username)
    .Filter("password_hash", Postgrest.Constants.Operator.Equals, pass)
    .Get();

var user = response.Models.FirstOrDefault();
```
* **Paso a paso de lo que hace el código al tocar "Iniciar Sesión":**
  1. Toma lo que el usuario escribió en `TxtUsuario` y `TxtPassword`, quitando espacios vacíos en los bordes con `.Trim()`.
  2. Verifica con `string.IsNullOrWhiteSpace` que no haya dejado las cajas vacías y que la clave no supere los 6 caracteres.
  3. Enciende la ruedita de carga (`LoadingIndicator.IsRunning = true`).
  4. Consulta a la tabla `usuarios` en Supabase filtrando donde `username` y `password_hash` coincidan exactamente (`Operator.Equals`).
  5. `FirstOrDefault()` toma el primer usuario encontrado (o devuelve `null` si no existe).
  6. Si existe, lo guarda en `SessionService.CurrentUser = user` y revisa su rol: si es `"admin"` reemplaza toda la pantalla principal por `new AdminShell()`, y si es `"usuario"` la reemplaza por `new UserShell()`.
  7. El bloque `finally` apaga siempre la ruedita de carga, haya salido bien o haya dado error.

---

### 6.3. Lógica de `RegisterPage.xaml.cs` y `ForgotPasswordPage.xaml.cs`
*(La estructura visual y de conexión es igual a la de Login, solo cambia la operación en base de datos):*
* **En Registro (`RegisterPage`):**
  1. Primero hace un `.Get()` buscando si ya existe alguien con ese `username`. Si `checkRes.Models.Count > 0`, avisa *"El nombre de usuario ya está registrado"*.
  2. Si está libre, crea un objeto `new Usuario { ... Rol = "usuario" }` y ejecuta `await _supabaseService.Client.From<Usuario>().Insert(nuevoUsuario);` (que equivale a un `INSERT INTO` en SQL).
* **En Recuperar Contraseña (`ForgotPasswordPage`):**
  1. Busca al usuario por su `username`.
  2. Si lo encuentra, le cambia la propiedad `user.PasswordHash = pass;` y ejecuta `.Update(user)` (que equivale a un `UPDATE` en SQL).

---

## CAPÍTULO 7: Módulo del Cliente (`UserMisTurnosPage`, `UserAgregarTurnoPage`, `UserHistorialPage`)

### 7.1. `UserMisTurnosPage.xaml.cs` (Panel del Cliente y Alerta de Próximo Service)
Esta pantalla se ejecuta automáticamente al entrar como cliente gracias al método **`OnAppearing()`** (que es un evento de .NET MAUI que se dispara cada vez que la pantalla aparece en el celular).

#### A. ¿Cómo funciona la Alerta Inteligente de Próximo Service?
```csharp
var hoy = DateTime.Today;
var limiteAlerta = hoy.AddDays(15);

_turnoProximoService = res.Models
    .Where(t => t.FechaProximoService.HasValue && t.FechaProximoService.Value.Date <= limiteAlerta)
    .OrderBy(t => t.FechaProximoService)
    .FirstOrDefault();
```
* **Explicación para defender:**
  1. Calculamos la fecha límite sumándole 15 días a la fecha de hoy (`hoy.AddDays(15)`).
  2. Con **LINQ** (`.Where(...)`) buscamos entre los turnos de este cliente si alguno tiene cargada una `FechaProximoService` que sea menor o igual a dentro de 15 días (es decir, que esté por vencer pronto o ya haya vencido).
  3. Si encuentra uno, hace visible la tarjeta dorada (`FrameAlertaService.IsVisible = true`) mostrando la patente y la fecha.
  4. Cuando el cliente toca esa tarjeta dorada, se ejecuta `OnAgendarServiceAlertaClicked`, que empaqueta los datos del auto en un `Dictionary<string, object>` y los envía a la pantalla de pedir turno para que ya aparezcan escritos.

#### B. ¿Cómo se muestran los Turnos Activos abajo del menú?
* En el XAML usamos un **`<CollectionView>`**, que es el componente de .NET MAUI para dibujar listas dinámicas.
* Dentro de `<CollectionView.ItemTemplate>` usamos **`{Binding Patente}`**, **`{Binding MarcaVehiculo}`**, **`{Binding Estado}`**, etc.
* **¿Qué significa `{Binding ...}`?** Es el *Enlace de Datos*. Conecta cada etiqueta de la tarjeta visual con la propiedad del mismo nombre dentro de la clase `Turno.cs`.

---

### 7.2. `UserAgregarTurnoPage.xaml.cs` (Solicitud de Turno y Regla de Máximo 3 Autos)

#### A. Recepción de datos automáticos (`[QueryProperty]`)
```csharp
[QueryProperty(nameof(PatenteParam), "Patente")]
```
* Este atributo recibe los datos cuando el cliente tocó la alerta dorada de Service y rellena las cajas de texto automáticamente para ahorrarle trabajo. Además, toma el nombre y teléfono desde `SessionService.CurrentUser` y los pone en cajas de solo lectura (`IsReadOnly="True"`).

#### B. Validación del Límite de 3 Vehículos por Fecha
```csharp
var fechaStr = fechaElegida.ToString("yyyy-MM-dd");
var turnosDelDia = await _supabaseService.Client
    .From<Turno>()
    .Filter("fecha_turno", Postgrest.Constants.Operator.Equals, fechaStr)
    .Get();

var turnosValidos = turnosDelDia.Models
    .Where(t => t.Estado != "Cancelado")
    .ToList();

if (turnosValidos.Count >= 3)
{
    await DisplayAlert("Cupo Completo", "Ya se alcanzó el límite máximo de 3 vehículos...", "Aceptar");
    return;
}
```
* **Explicación paso a paso:**
  1. Convierte la fecha elegida en el calendario (`DatePicker`) al formato universal de base de datos `yyyy-MM-dd` (Año-Mes-Día).
  2. Trae de Supabase todos los turnos registrados en esa fecha.
  3. Filtra con `.Where(t => t.Estado != "Cancelado")` porque si un turno fue cancelado, ese lugar quedó libre en el taller.
  4. Si la cantidad (`Count`) es mayor o igual a `3`, muestra el cartel de *"Cupo Completo"* y corta la ejecución con `return;` sin guardar nada.
  5. Si hay menos de 3, inserta el nuevo turno con `Estado = "Solicitado"`.

---

### 7.3. `UserHistorialPage.xaml.cs` (Historial del Cliente y Descarga de Comprobante)
*(La carga de la lista con `CollectionView` es igual a `UserMisTurnosPage`, con la diferencia de que aquí filtra `.Where(t => t.Estado == "Finalizado" || t.Estado == "Cancelado")`)*:
* **Lo nuevo aquí es el botón de Comprobante:**
  Cuando el cliente toca **"Descargar / Ver Comprobante"**, llama a `PdfService.GenerarComprobanteTurnoPdfAsync(...)` y luego ejecuta:
  ```csharp
  await Share.Default.RequestAsync(new ShareFileRequest
  {
      Title = $"Comprobante - {t.Patente}",
      File = new ShareFile(rutaArchivo)
  });
  ```
  `Share.Default.RequestAsync` es una herramienta nativa de `Microsoft.Maui.ApplicationModel.DataTransfer` que abre el menú oficial de Android o Windows para enviar el archivo por WhatsApp, guardarlo en el teléfono o subirlo a Drive.

---

## CAPÍTULO 8: Módulo del Administrador (`AdminTurnosActivosPage`, `AdminDetalleTurnoPage`, `AdminRepuestosPage`, `AdminVentaRepuestosPage`, `AdminBalancePage`)

### 8.1. `AdminTurnosActivosPage.xaml.cs` (Dashboard, Desplegables y Buscador)

#### A. Cruce de Tablas en Memoria (`ToDictionary`)
```csharp
var turnosRes = await _supabaseService.Client.From<Turno>().Get();
var usuariosRes = await _supabaseService.Client.From<Usuario>().Get();
var dictUsuarios = usuariosRes.Models.ToDictionary(u => u.Id, u => u);

foreach (var t in todos)
{
    if (dictUsuarios.TryGetValue(t.UsuarioId, out var user))
    {
        t.NombreCliente = $"{user.Nombre} {user.Apellido}";
        t.TelefonoCliente = user.Telefono;
    }
}
```
* **¿Qué hace este bloque?** Como la tabla `turnos` solo tiene el número `usuario_id`, traemos la lista de usuarios y la convertimos en un **Diccionario de búsqueda rápida (`ToDictionary`)** donde la clave es el `Id`. Luego recorremos los turnos y le pegamos a cada uno el nombre, apellido y teléfono de su dueño.

#### B. Separación en dos bandejas y Contadores
* Divide la lista en dos:
  1. `_solicitadosOriginal`: los que tienen `Estado == "Solicitado"`.
  2. `_activosOriginal`: los que tienen `Estado == "Aceptado"`, `"Reprogramado"` o `"En Proceso"`.
* Actualiza los numeritos de las insignias: `LblCountSolicitados.Text = _solicitadosOriginal.Count.ToString();`.

#### C. ¿Cómo funcionan las barras desplegables (Acordeones)?
```csharp
private void OnToggleSolicitadosClicked(object? sender, EventArgs e)
{
    PanelSolicitados.IsVisible = !PanelSolicitados.IsVisible;
    LblFlechaSolicitados.Text = PanelSolicitados.IsVisible ? "▼" : "▲";
}
```
* El signo `!` invierte el valor booleano: si `IsVisible` era `true` (visible), pasa a `false` (oculto), y viceversa. Además cambia la flechita entre `▼` y `▲`.

#### D. ¿Cómo funciona el Buscador en Tiempo Real? *(Se repite igual en Historial, Catálogo y Ventas)*
```csharp
private void OnBuscarTurnosChanged(object sender, TextChangedEventArgs e)
{
    FiltrarListas(e.NewTextValue);
}
```
* Cada vez que el usuario escribe o borra una letra en el `<SearchBar>`, se dispara el evento `TextChanged`.
* El método convierte lo escrito a minúsculas (`.ToLower()`) y busca con `.Contains(f)` si ese texto está adentro de la `Patente`, de la `MarcaVehiculo`, del `ModeloVehiculo` o del `NombreCliente`, actualizando la lista en el acto sin volver a consultar a internet.

---

### 8.2. `AdminDetalleTurnoPage.xaml.cs` (Aceptación, Reprogramación, Diagnóstico y Repuestos)

1. **Ocultamiento inteligente de los botones Aceptar y Reprogramar:**
   ```csharp
   PanelAccionesTurno.IsVisible = TurnoActual.Estado == "Solicitado";
   ```
   Si el turno ya fue aceptado antes, el panel de botones ni siquiera aparece. Y cuando el mecánico toca **"✔ Aceptar Turno"**, cambia el estado a `"Aceptado"` y pone `PanelAccionesTurno.IsVisible = false;` para que desaparezcan en ese mismo instante.

2. **Asignación de Repuestos y Descuento de Stock:**
   Cuando el mecánico elige un repuesto del desplegable (`PickerRepuestos`) y toca **"Agregar Repuesto"**:
   * Revisa que `cantidad <= repuesto.Stock`.
   * Calcula `subtotal = cantidad * repuesto.PrecioUnitario`.
   * Inserta el registro en la tabla intermedia `turno_repuestos`.
   * Le resta la cantidad al stock del repuesto (`repuesto.Stock -= cantidad`) y hace `.Update(repuesto)` en Supabase.

3. **Cálculo Automático del Costo Total:**
   ```csharp
   private void ActualizarCostoTotal()
   {
       decimal.TryParse(TxtManoObra.Text, out decimal manoObra);
       decimal sumaRepuestos = _repuestosUsados.Sum(r => r.Subtotal);
       decimal total = manoObra + sumaRepuestos;
       LblCostoTotal.Text = $"Costo Total: ${total:N2}";
   }
   ```
   Cada vez que el mecánico escribe un número en la caja de Mano de Obra (`OnManoObraChanged`) o agrega un repuesto, este método suma la mano de obra más todos los subtotales de la lista `_repuestosUsados` usando `.Sum()` y lo muestra formateado con 2 decimales (`:N2`).

4. **Guardado de Fecha de Finalización y Próximo Service:**
   Al tocar **"Guardar Cambios"**, si el estado elegido es `"Finalizado"`, guarda `TurnoActual.FechaFinalizacion = DateTime.UtcNow;`. Y si el turno era de tipo `"Service"`, guarda también `TurnoActual.FechaProximoService = DpProximoService.Date;`.

---

### 8.3. `AdminRepuestosPage.xaml.cs` (Catálogo CRUD de Repuestos)
*(El buscador y la lista funcionan igual que en las páginas anteriores. Aquí explicamos cómo hace Alta, Modificación y Baja con un mismo formulario):*
* **Variable bandera `_repuestoSeleccionado`:**
  * Cuando el administrador llena Nombre, Stock y Precio y toca **"Guardar"**, el código mira si `_repuestoSeleccionado == null`.
  * Si es `null`, significa que está creando un repuesto nuevo -> ejecuta **`.Insert(nuevo)`**.
  * Si no es `null` (porque antes tocó el botón amarillo **"Editar"** de un repuesto de la lista), actualiza los datos de ese repuesto -> ejecuta **`.Update(_repuestoSeleccionado)`**.
* **Eliminación segura (`OnEliminarRepuestoClicked`):**
  * Ejecuta `.Delete(r)`. Si ese repuesto ya había sido usado en algún vehículo, la base de datos bloquea el borrado gracias a la regla `ON DELETE RESTRICT` de SQL y el bloque `catch` muestra el aviso: *"No se puede eliminar porque este repuesto ya fue utilizado en un turno"*, cuidando que no se rompa el historial contable.

---

### 8.4. `AdminVentaRepuestosPage.xaml.cs` (Venta de Mostrador)
*(El buscador y el descuento de stock en Supabase funcionan igual que en los puntos anteriores. Explicamos los dos detalles técnicos exclusivos de esta pantalla):*

1. **Separación entre `_todosItemsVenta` e `ItemsMostrados`:**
   * `_todosItemsVenta` guarda en memoria todos los repuestos del taller con las cantidades que el vendedor va escribiendo.
   * `ItemsMostrados` es una `ObservableCollection` que muestra solo los que coinciden con el buscador.
   * Gracias a esto, `CalcularTotal()` suma sobre `_todosItemsVenta.Sum(i => i.Subtotal)`, logrando que aunque busques un repuesto y luego busques otro, **nunca se borre lo que ya habías cargado en el carrito de venta**.
2. **Uso de `MainThread.BeginInvokeOnMainThread`:**
   ```csharp
   MainThread.BeginInvokeOnMainThread(() =>
   {
       ItemsMostrados.Clear();
       foreach (var item in resultado) ItemsMostrados.Add(item);
       CalcularTotal();
   });
   ```
   * **¿Qué significa esto?** En Android, solo el "Hilo Principal" (*Main Thread*) tiene permiso para tocar o redibujar controles de la pantalla. Envolver el refresco de la lista dentro de `BeginInvokeOnMainThread` evita el error nativo `Cannot call removeView(At)` cuando se recarga la pantalla después de vender.

---

### 8.5. `AdminBalancePage.xaml.cs` (Balance del Mes Actual e Historial de Meses)
* **¿Cómo calcula las métricas del mes actual?**
  1. Obtiene el primer día del mes actual (`new DateTime(hoy.Year, hoy.Month, 1)`) y el último día del mes (`inicioMesActual.AddMonths(1).AddDays(-1)`).
  2. Filtra los turnos finalizados dentro de ese rango de fechas y suma su `CostoTotal` con `.Sum(t => t.CostoTotal)` -> eso da los **Ingresos por Vehículos**.
  3. Filtra las ventas de mostrador (`VentaRepuesto`) de ese mismo mes y suma su `Subtotal` -> eso da los **Ingresos por Mostrador**.
  4. Suma ambos valores para mostrar el número verde grande (**Total de Dinero Obtenido**).
* **¿Cómo genera automáticamente los cuadros de cada mes abajo del todo?**
  1. Junta todas las fechas de turnos y de ventas, extrae los pares únicos de `{ Año, Mes }` con `.Distinct()` y los ordena del más reciente al más antiguo.
  2. Recorre cada mes con un `foreach`, calcula cuánto se ganó en vehículos, cuánto en repuestos y cuántos autos se atendieron en ese mes específico, y crea un objeto `BalanceMesItem` con el nombre del mes en español (`new CultureInfo("es-ES")`, ej: *"Noviembre 2026"*).

---

## CAPÍTULO 9: Posibles Preguntas del Profesor en la Defensa (y cómo responderlas)

1. **Profesor: *"¿Qué patrón o estructura usaron para organizar el proyecto?"***
   * **Tu respuesta:** *"Usamos una arquitectura modular en capas separada en cuatro carpetas: `Models` (las clases que representan las tablas de la base de datos), `Services` (la lógica de conexión a Supabase, manejo de sesión y creación de comprobantes), `Views` (las pantallas XAML con su código C# de eventos) y `Converters` (para transformar estados en colores visuales)."*

2. **Profesor: *"¿Cómo hace la aplicación para saber si el que entró es Administrador o Cliente?"***
   * **Tu respuesta:** *"En la tabla `usuarios` tenemos la columna `rol`. Cuando el usuario inicia sesión en `LoginPage`, consultamos Supabase y guardamos el usuario en `SessionService.CurrentUser`. Ahí evaluamos con un `if` si `user.Rol` es igual a `'admin'`. Si es admin, cargamos `AdminShell`; si es `'usuario'`, cargamos `UserShell`."*

3. **Profesor: *"¿Cómo hacen para pasar los datos de un turno cuando tocan 'Gestionar Turno' hacia la otra pantalla?"***
   * **Tu respuesta:** *"Usamos el sistema de paso de parámetros de Shell (`Shell.Current.GoToAsync`). Creamos un diccionario (`Dictionary<string, object>`) donde guardamos el objeto `Turno` seleccionado, y en la pantalla de destino (`AdminDetalleTurnoPage`) lo recibimos automáticamente arriba de la clase con el atributo `[QueryProperty(nameof(TurnoActual), "Turno")]`."*

4. **Profesor: *"¿Cómo controlan que no se venda más stock del que hay disponible?"***
   * **Tu respuesta:** *"Tenemos una doble validación: primero, en las listas de venta y de asignación solo cargamos repuestos que tengan `Stock > 0`. Segundo, antes de guardar en Supabase verificamos con un `if` que la cantidad escrita no supere a `StockDisponible`. Recién cuando pasa esa validación, insertamos la venta y actualizamos la tabla `repuestos` restándole las unidades vendidas."*

5. **Profesor: *"¿Qué significa `async` y `await` que aparece en casi todos los métodos?"***
   * **Tu respuesta:** *"Significa programación asíncrona. Como nuestra base de datos está en internet (en Supabase), cada vez que pedimos o guardamos datos hay una pequeña demora de red. Usar `async` y `await` hace que el teléfono espere la respuesta de internet en segundo plano sin congelar ni trabar la pantalla del usuario."*
