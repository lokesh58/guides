1. enabled dev mode
2. installed python install manager
3. added path for pim ^ & another change was done to allow big paths, not sure about it

scoop packages (corresponding winget packages):
1. 7zip (7zip.7zip)
2. ast-grep (ast-grep.ast-grep)
3. cmake (Kitware.CMake)
4. fd (sharkdp.fd)
5. git (Git.Git)
6. gzip (xxx not needed)
7. imagemagick (ImageMagick.ImageMagick)
8. lazygit (JesseDuffield.lazygit)
9. Meslo-NF
10. mingw (xxx not installing this - use microsoft compiler instead)
11. neovim (Neovim.Neovim)
12. ninja (xxx not installing this - use the nmake system instead)
13. ripgrep (BurntSushi.ripgrep.MSVC)
14. unzip (xxx not needed)
15. volta (Volta.Volta)
16. wget (xxx not needed)

final winget packages:
1. Git.Git
2. Neovim.Neovim
3. 7zip.7zip (required for mason)
4. GnuWin32.Tar (required for mason)
5. Python.PythonInstallManager (python required by mason for installations)
6. Volta.Volta (node.js required by mason for installations)
7. sharkdp.fd (used by snacks)
8. BurntSushi.ripgrep.MSVC (used by snacks)
9. JesseDuffield.lazygit (used by snacks - and good in general)
10. ImageMagick.ImageMagick (used by snacks)
11. ast-grep.ast-grep (used by find and replace plugin neovim)
12. Kitware.CMake
13. BrechtSanders.WinLibs.POSIX.UCRT.LLVM
