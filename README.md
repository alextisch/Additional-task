# Additional-task
# Отчет по коду GitHub Actions workflow "Build Packages"

## Общее описание
Данный workflow автоматизирует процесс сборки проекта Xonix (игры) для Linux и Windows, создавая пакеты для каждого из этих окружений. Workflow запускается при push и pull request в ветку main.

## Структура workflow

### Триггеры (on)
```yaml
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
```
- Запускает workflow при:
  - Push в ветку main
  - Создании pull request в ветку main

### Задачи (jobs)

#### 1. Сборка для Linux (build-linux)
Выполняется на Ubuntu последней версии.

**Шаги:**

1. **Checkout code**
   ```yaml
   uses: actions/checkout@v4
   ```
   - Клонирует репозиторий в среду выполнения workflow

2. **Install build dependencies**
   ```yaml
   run: |
     sudo apt-get update
     sudo apt-get install -y \
       libsfml-dev \
       cmake \
       build-essential \
       debhelper \
       dh-make \
       fakeroot
   ```
   - Обновляет список пакетов
   - Устанавливает необходимые зависимости:
     - libsfml-dev - библиотеки SFML для разработки
     - cmake - система сборки
     - build-essential - базовые инструменты компиляции
     - debhelper, dh-make, fakeroot - инструменты для создания DEB-пакетов

3. **Configure and build**
   ```yaml
   run: |
     mkdir -p build
     cd build
     cmake -DCMAKE_BUILD_TYPE=Release ..
     make -j$(nproc)
   ```
   - Создает директорию build
   - Запускает cmake с конфигурацией Release
   - Запускает сборку с максимальным использованием ядер процессора

4. **Create DEB package**
   ```yaml
   run: |
     # Создает структуру пакета
     mkdir -p pkg/DEBIAN pkg/usr/games
     cp build/xonix pkg/usr/games/

     # Создает control файл
     cat > pkg/DEBIAN/control <<EOF
     Package: xonix
     Version: 1.0-0
     Section: games
     Priority: optional
     Architecture: amd64
     Maintainer: Your Name <your.email@example.com>
     Depends: libsfml-system2.5, libsfml-window2.5, libsfml-graphics2.5
     Description: Xonix game
      A classic Xonix game built with SFML.
     EOF

     # Собирает пакет
     dpkg-deb --build pkg xonix_1.0-0_amd64.deb
   ```
   - Создает структуру DEB-пакета
   - Копирует бинарник в нужную директорию
   - Генерирует control-файл с метаданными пакета
   - Собирает DEB-пакет с помощью dpkg-deb

5. **Upload DEB artifact**
   ```yaml
   uses: actions/upload-artifact@v4
   with:
     name: xonix-deb-package
     path: xonix_1.0-0_amd64.deb
   ```
   - Загружает собранный DEB-пакет как артефакт workflow

#### 2. Сборка для Windows (build-windows)
Выполняется на Windows последней версии.

**Шаги:**

1. **Checkout code**
   ```yaml
   uses: actions/checkout@v4
   ```
   - Аналогично Linux-версии, клонирует репозиторий

2. **Download and install SFML**
   ```powershell
   run: |
     $url = "https://github.com/SFML/SFML/releases/download/2.5.1/SFML-2.5.1-windows-vc15-64-bit.zip"
     $output = "$env:RUNNER_TEMP\SFML-2.5.1.zip"
     Invoke-WebRequest -Uri $url -OutFile $output -UserAgent "GitHubActions"
     $sfmlPath = "C:\SFML-2.5.1"
     Expand-Archive -Path $output -DestinationPath $sfmlPath -Force
     Write-Output "SFML_DIR=$sfmlPath" | Out-File -FilePath $env:GITHUB_ENV -Append
     Write-Output "CMAKE_PREFIX_PATH=$sfmlPath" | Out-File -FilePath $env:GITHUB_ENV -Append
     Write-Output "PATH=$env:PATH;$sfmlPath\bin" | Out-File -FilePath $env:GITHUB_ENV -Append
   ```
   - Скачивает SFML библиотеки для Windows
   - Распаковывает их в C:\SFML-2.5.1
   - Устанавливает переменные окружения для последующих шагов

3. **Build with CMake**
   ```powershell
   run: |
     mkdir build
     cd build
     cmake -G "Visual Studio 17 2022" -A x64 ..
     cmake --build . --config Release --target xonix
   ```
   - Создает директорию build
   - Генерирует проект Visual Studio 2022 для 64-битной архитектуры
   - Собирает Release-версию проекта

4. **Create MSI package**
   ```powershell
   run: |
     $productGuid = [guid]::NewGuid().ToString().ToUpper()
     $componentGuid = [guid]::NewGuid().ToString().ToUpper()
     @"
     <?xml version="1.0" encoding="UTF-8"?>
     <Wix xmlns="http://schemas.microsoft.com/wix/2006/wi">
       <Product Id="*" Name="Xonix" Language="1033" Version="1.0.0.0" Manufacturer="YourCompany" UpgradeCode="$productGuid">
         <Package InstallerVersion="200" Compressed="yes" Comments="Windows Installer Package" Platform="x64"/>
         <MajorUpgrade DowngradeErrorMessage="A newer version of [ProductName] is already installed." />
         <MediaTemplate EmbedCab="yes"/>
       
         <Directory Id="TARGETDIR" Name="SourceDir">
           <Directory Id="ProgramFiles64Folder">
             <Directory Id="INSTALLFOLDER" Name="Xonix">
               <Component Id="ApplicationFiles" Guid="$componentGuid" Win64="yes">
                 <File Id="XonixEXE" Name="xonix.exe" Source="build\Release\xonix.exe" KeyPath="yes"/>
               </Component>
             </Directory>
           </Directory>
         </Directory>
       
         <Feature Id="MainApplication" Title="Main Application" Level="1">
           <ComponentRef Id="ApplicationFiles"/>
         </Feature>
       </Product>
     </Wix>
     "@ | Out-File -FilePath product.wxs -Encoding utf8
     & "C:\Program Files (x86)\WiX Toolset v3.14\bin\candle.exe" -arch x64 -out product.wixobj product.wxs
     & "C:\Program Files (x86)\WiX Toolset v3.14\bin\light.exe" -out xonix.msi product.wixobj
   ```
   - Генерирует уникальные GUID для пакета
   - Создает WXS-файл (WiX Source) с конфигурацией установщика
   - Компилирует WXS в MSI-пакет с помощью WiX Toolset

5. **Upload MSI artifact**
   ```yaml
   uses: actions/upload-artifact@v4
   with:
     name: xonix-msi-package
     path: xonix.msi
   ```
   - Загружает собранный MSI-пакет как артефакт workflow

## Итог
Данный workflow автоматически собирает проект Xonix для двух платформ:
1. Для Linux - создает DEB-пакет
2. Для Windows - создает MSI-установщик

Результаты сборки сохраняются как артефакты workflow и могут быть скачаны после завершения выполнения.
