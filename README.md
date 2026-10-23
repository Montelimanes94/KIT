# ============================================================
# MODULO: Comun.ps1
# Funciones comunes de consola, validacion, lectura y utilidades de ficheros/procesos.
# ============================================================

# SALIDA POR CONSOLA
# ============================================================
# Descripcion: Muestra un bloque de cabecera para separar visualmente fases o menus.
# Parametros de entrada: Message.
function Write-Step {
  param([string]$Message)

  Write-Host ""
  Write-Host "==================================================" -ForegroundColor Cyan
  Write-Host $Message -ForegroundColor Cyan
  Write-Host "==================================================" -ForegroundColor Cyan
}
# Descripcion: Muestra un mensaje informativo con formato discreto.
# Parametros de entrada: Message.
function Write-Info {
  param([string]$Message)
  Write-Host "    $Message" -ForegroundColor Gray
}
# Descripcion: Muestra una advertencia en consola.
# Parametros de entrada: Message.
function Write-WarnText {
  param([string]$Message)
  Write-Host "    $Message" -ForegroundColor Yellow
}
# Descripcion: Muestra un mensaje de error en consola.
# Parametros de entrada: Message.
function Write-ErrorText {
  param([string]$Message)
  Write-Host "    $Message" -ForegroundColor Red
}
# Descripcion: Muestra la accion seleccionada por el usuario.
# Parametros de entrada: Message.
function Write-ActionPressed {
  param([string]$Message)

  Write-Host ""
  Write-Host "[ACCION] $Message" -ForegroundColor Green
}

# ============================================================
# UTILIDADES BASE
# ============================================================
# Descripcion: Elimina el fichero .yo-rc.json residual que Yeoman deja en la carpeta de proyectos.
function Remove-ProjectsRootYoRc {

  $rootYoRc = Join-Path $ProjectsRoot ".yo-rc.json"

  if (Test-Path -LiteralPath $rootYoRc) {

    Remove-Item `
      -LiteralPath $rootYoRc `
      -Force `
      -ErrorAction SilentlyContinue

    Write-Info "Eliminado .yo-rc.json de la carpeta proyectos"
  }
}
# Descripcion: Crea una carpeta si no existe.
# Parametros de entrada: Path.
function Ensure-Directory {
  param([string]$Path)

  if (-not (Test-Path -LiteralPath $Path -PathType Container)) {
    New-Item -ItemType Directory -Path $Path -Force | Out-Null
  }
}
# Descripcion: Lanza error si no existe un fichero.
# Parametros de entrada: Path, Label.
function Throw-IfMissingFile {
  param(
    [string]$Path,
    [string]$Label
  )

  if (-not (Test-Path -LiteralPath $Path -PathType Leaf)) {
    throw "No se encontro $Label en: $Path"
  }
}
# Descripcion: Lanza error si no existe una carpeta.
# Parametros de entrada: Path, Label.
function Throw-IfMissingDirectory {
  param(
    [string]$Path,
    [string]$Label
  )

  if (-not (Test-Path -LiteralPath $Path -PathType Container)) {
    throw "No se encontro $Label en: $Path"
  }
}
# Descripcion: Comprueba si tar esta disponible en el sistema.
# Salida: Booleano que indica si tar esta disponible.
function Test-TarAvailable {
  $tarCmd = Get-Command tar -ErrorAction SilentlyContinue
  return ($null -ne $tarCmd)
}
# Descripcion: Valida que el nombre del proyecto sea un nombre de carpeta valido.
# Parametros de entrada: Name.
# Salida: Booleano que indica si el nombre del proyecto es valido.
function Test-ProjectNameValid {
  param([string]$Name)

  if ([string]::IsNullOrWhiteSpace($Name)) {
    return $false
  }

  if ($Name -in @(".", "..")) {
    return $false
  }

  if ($Name.Contains("\") -or $Name.Contains("/")) {
    return $false
  }

  $invalidChars = [System.IO.Path]::GetInvalidFileNameChars()

  foreach ($ch in $invalidChars) {
    if ($Name.Contains([string]$ch)) {
      return $false
    }
  }

  return $true
}
# Descripcion: Lee una opcion numerica de menu y valida que este dentro del rango permitido.
# Parametros de entrada: Prompt, Min, Max.
# Salida: Numero de opcion seleccionado por el usuario.
function Read-MenuNumber {
  param(
    [string]$Prompt,
    [int]$Min,
    [int]$Max
  )

  while ($true) {
    $answer = Read-Host $Prompt
    $selected = 0

    if ([int]::TryParse($answer, [ref]$selected)) {
      if ($selected -ge $Min -and $selected -le $Max) {
        return $selected
      }
    }

    Write-WarnText "Seleccion invalida. Introduce solo un numero entre $Min y $Max."
  }
}
# Descripcion: Lee una respuesta si/no con soporte para valor por defecto.
# Parametros de entrada: Prompt, DefaultYes.
# Salida: Booleano con la respuesta del usuario.
function Read-YesNo {
  param(
    [string]$Prompt,
    [switch]$DefaultYes
  )

  $hint = if ($DefaultYes) { "[S/n]" } else { "[s/N]" }
  $answer = Read-Host "$Prompt $hint"

  if ($null -eq $answer) {
    $answer = ""
  }

  $answer = $answer.Trim().ToLowerInvariant()

  if ([string]::IsNullOrWhiteSpace($answer)) {
    return [bool]$DefaultYes
  }

  return $answer -in @("s", "si", "y", "yes")
}
# Descripcion: Escribe un fichero de texto en UTF-8 sin BOM.
# Parametros de entrada: Path, Content.
function Set-TextFileUtf8NoBom {
  param(
    [string]$Path,
    [string]$Content
  )

  $utf8NoBom = New-Object System.Text.UTF8Encoding -ArgumentList $false
  [System.IO.File]::WriteAllText($Path, $Content, $utf8NoBom)
}
# Descripcion: Asegura permisos de escritura en las carpetas generadas de un proyecto SPFx.
function Ensure-ProjectWritableOutputs {
  param([string]$ProjectPath)

  $currentIdentity = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
  $outputFolders = @("src", "lib")

  foreach ($outputFolder in $outputFolders) {
    $outputPath = Join-Path $ProjectPath $outputFolder

    if (-not (Test-Path -LiteralPath $outputPath -PathType Container)) {
      continue
    }

    try {
      & icacls $outputPath /grant "${currentIdentity}:(OI)(CI)M" /T /C | Out-Null
      if ($LASTEXITCODE -ne 0) {
        Write-WarnText "No se pudieron ajustar todos los permisos de $outputPath. Se continuara con los permisos actuales."
      }
    }
    catch {
      Write-WarnText "No se pudo ejecutar icacls: $($_.Exception.Message). Se continuara con los permisos actuales."
      return
    }
  }
}
# Descripcion: Elimina comentarios // y /* */ de un JSONC respetando cadenas de texto.
# Parametros de entrada: Content.
# Salida: Texto JSON sin comentarios.
function Remove-JsonComments {
  param([string]$Content)

  $result = New-Object System.Text.StringBuilder
  $inString = $false
  $escape = $false
  $i = 0

  while ($i -lt $Content.Length) {
    $char = $Content[$i]

    if ($escape) {
      [void]$result.Append($char)
      $escape = $false
      $i++
      continue
    }

    if ($char -eq [char]92) {
      [void]$result.Append($char)
      $escape = $true
      $i++
      continue
    }

    if ($char -eq [char]34) {
      [void]$result.Append($char)
      $inString = -not $inString
      $i++
      continue
    }

    if (-not $inString -and $char -eq [char]47 -and ($i + 1) -lt $Content.Length) {
      $next = $Content[$i + 1]

      if ($next -eq [char]47) {
        while ($i -lt $Content.Length -and $Content[$i] -ne [char]10) {
          $i++
        }
        continue
      }

      if ($next -eq [char]42) {
        $i += 2

        while (($i + 1) -lt $Content.Length -and -not ($Content[$i] -eq [char]42 -and $Content[$i + 1] -eq [char]47)) {
          $i++
        }

        if (($i + 1) -lt $Content.Length) {
          $i += 2
        }

        continue
      }
    }

    [void]$result.Append($char)
    $i++
  }

  return $result.ToString()
}
# Descripcion: Lee un fichero JSON/JSONC eliminando comentarios antes de convertirlo.
# Parametros de entrada: Path.
# Salida: Objeto PowerShell obtenido a partir del fichero JSON/JSONC.
function ConvertFrom-JsoncFile {
  param([string]$Path)

  $raw = Get-Content -LiteralPath $Path -Raw
  $clean = Remove-JsonComments -Content $raw
  return $clean | ConvertFrom-Json
}
# Descripcion: Ejecuta un comando externo en un directorio concreto.
# Parametros de entrada: true.
function Invoke-ExternalCommand {
  param(
    [Parameter(Mandatory = $true)]
    [string]$FilePath,

    [Parameter(Mandatory = $true)]
    [string[]]$Arguments,

    [Parameter(Mandatory = $true)]
    [string]$WorkingDirectory
  )

  Write-Host ""
  Write-Host ">>Ejecutando: $FilePath $($Arguments -join ' ')" -ForegroundColor Cyan
  Write-Host "   WorkingDirectory: $WorkingDirectory" -ForegroundColor DarkGray

  if (Test-Path -LiteralPath (Join-Path $WorkingDirectory "package.json") -PathType Leaf) {
    Ensure-ProjectWritableOutputs -ProjectPath $WorkingDirectory
  }

  Push-Location $WorkingDirectory
  try {
    & $FilePath @Arguments

    if ($LASTEXITCODE -ne 0) {
      throw "El comando devolvio codigo $LASTEXITCODE"
    }
  }
  finally {
    Pop-Location
  }
}

# ============================================================
