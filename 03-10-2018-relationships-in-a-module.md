# Как увидеть граф связей внутри вашего PowerShell модуля

День добрый, коллеги. Сто лет назад где-то описывал быстрый и "грязный" способ увидеть граф вызовов внутри вашего PowerShell скрипта. Теперь, пришла, так сказать, пора натянуть сову на глобус и посмотреть граф зависимостей внутри модуля.

Прежде всего, нудно установить [PSQuickGraph module](https://www.powershellgallery.com/packages/PSQuickGraph/1.1) из Powershell Gallery. И следом за этим запустить скрипт, базу которого можно взять [вот тут](https://gist.github.com/eosfor/e66b1bb2c4d7fa5279dc1ffbcfd6f205). Честно говоря, я не совсем понял, как вставить код из gist в пост на хабре. Если кто подскажет в коментах, буду премного благодарен.

Итак, продолжим. Прежде всего, в двух словал про сам модуль. Он строится на библиотеке [QuickGraph](https://archive.codeplex.com/?p=quickgraph). По сути, это PowerShell обертка над библиотекой, которая предназначена для быстрых и грубых тестов с графами. Одним из таких тестов может быть штука, которая разбирает ваш PowerShell код, генерирует из него граф связей, и представляет его визуально.

Кроме того, в самом скрипте используется встроенный в PowerShell механизм доступа к его [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree) что, собственно, значительно упрощает нам разбор кода. В конечном итоге скомбинировав одно и другое мы получаем граф.

В итоге, сам скрипт состоит из двух частей. Куска "парсера", работающего с AST, и части, которая соственно связывает AST и граф. Эта часть описана на следующем примере

```powershell
# создаем экземпляр графа (модуль PSQuickGraph)
$g = New-Graph -Type BidirectionalGraph
# парсим PS1 файлы, выгребаем AST и добавляем функции как вершины в граф
$f = dir "C:\Repo\Work\Tools\*.ps1" -Recurse | Find-Function
$f.name | % { Add-Vertex -Vertex $_ -Graph $g}


# бежим второй раз, и добавляем вершины там, где происходят вызовы функций модуля и взяких других командлетов, которые нам могут быть интересны
$f | % {
    $fName = $_.name
    $tmpFile = New-TemporaryFile
    $_.line >> ($tmpFile.fullname)
    $expr = Find-Expression -Fullname $tmpFile.fullname
    $expr.name | ? {$_} | ? {$_ -notin "%","where","?","select","out-null","sort","ft","fl","Write-Verbose" } | % { Add-Edge -From $fName -to $_ -Graph $g}
}

# отображаем граф
Show-GraphLayout -Graph $g

# сохраняем его в формате DOT
Export-Graph -Format Graphviz -Graph $g -Path C:\Temp\g.gv
```

Ну и, примерно вот так это выглядит в результате

![moduleStructure](https://habrastorage.org/webt/my/5j/p5/my5jp5jvx4p9988bwxj9qbdtvqg.png)

Кроме этого, последняя команда экспортирует граф в формат DOT, что позволяет помучать его еще в приложении [graphviz](https://www.graphviz.org)