# Еще немного про графы, или как обнаружить зависимости между вашими приложениями

Доброе время суток, коллеги. Последнее время довольно много разговоров о переносе приложений из физических инфраструктур, читай датацентров, в облако. Например в [Microsoft Azure](https://azure.microsoft.com/en-us/). Ну, или вообще, о любом другом переносе одного или нескольких приложений из одного места в другое. Одной из самых больших проблем в такого рода задачах является необходимость найти все внешние зависимости приложения. Имеется в виду не зависимости в коде, а зависимости от внешних, по отношению к приложению, систем. Собственно говоря, порой нам надо найти, с кем наше предложение разговаривает, и кто разговаривает с ним. Как это сделать, если у нас нет развернутой SIEM, так сказать средствами "SIEM для бедных". Собственно говоря, для систем на Windows есть следующее предложение.

Нам нужно:

- Включить логирование в Windows Firewall на всех машинах, так или иначе ассоциированных с приложением
- Скачать на админскую станцию [PSQuickGraph module](https://www.powershellgallery.com/packages/PSQuickGraph/1.1)
- Собрать в кучу и проанализировать логи, построить граф связей

В простейшем виде выглядит анализ логов примерно вот так. Ужасно, в лоб, но что поделаешь. На самом деле я даже поленился написать логику для забега по логам, просто скопипастил все дважды. Но для наших целей "грубо и в лоб" тоже пойдет, дабы показать идею

```powershell
$f = gc "C:\Temp\pfirewall_public.log"
$regex = '^(?<datetime>\d{4,4}-\d{2,2}-\d{2,2}\s\d{2}:\d{2}:\d{2})\s(?<action>\w+)\s(?<protocol>\w+)\s(?<srcip>\b(?:\d{1,3}\.){3}\d{1,3}\b)\s(?<dstip>\b(?:\d{1,3}\.){3}\d{1,3}\b)\s(?<srcport>\d{1,5})\s(?<dstport>\d{1,5})\s(?<size>\d+|-)\s(?<tcpflags>\d+|-)\s(?<tcpsyn>\d+|-)\s(?<tcpack>\d+|-)\s(?<tcpwin>\d+|-)\s(?<icmptype>\d+|-)\s(?<icmpcode>\d+|-)\s(?<info>\d+|-)\s(?<path>.+)$'

$log =
$f | % {
    $_ -match $regex | Out-Null
    if ($Matches) {
    [PSCustomObject]@{
        action   = $Matches.action
        srcip    = [ipaddress]$Matches.srcip
        dstport  = $Matches.dstport
        tcpflags = $Matches.tcpflags
        dstip    = [ipaddress]$Matches.dstip
        info     = $Matches.info
        size     = $Matches.size
        protocol = $Matches.protocol
        tcpack   = $Matches.tcpac
        srcport  = $Matches.srcport
        tcpsyn   = $Matches.tcpsyn
        datetime = [datetime]$Matches.datetime
        icmptype = $Matches.icmptype
        tcpwin   = $Matches.tcpwin
        icmpcode = $Matches.icmpcode
        path     = $Matches.path
    }
    }
}

$f = gc "C:\Temp\pfirewall_public2.log"
$log2 =
$f | % {
    $_ -match $regex | Out-Null
    if ($Matches) {
    [PSCustomObject]@{
        action   = $Matches.action
        srcip    = [ipaddress]$Matches.srcip
        dstport  = $Matches.dstport
        tcpflags = $Matches.tcpflags
        dstip    = [ipaddress]$Matches.dstip
        info     = $Matches.info
        size     = $Matches.size
        protocol = $Matches.protocol
        tcpack   = $Matches.tcpac
        srcport  = $Matches.srcport
        tcpsyn   = $Matches.tcpsyn
        datetime = [datetime]$Matches.datetime
        icmptype = $Matches.icmptype
        tcpwin   = $Matches.tcpwin
        icmpcode = $Matches.icmpcode
        path     = $Matches.path
    }
    }
}

$l = $log + $log2

$g = new-graph -Type BidirectionalGraph
 
$l | ? {$_.srcip -and $_.dstip} | % {
    Add-Edge -From $_.srcip -To $_.dstip -Graph $g | out-null
}

Show-GraphLayout -Graph $g
```

Собственно говоря, в данном примере мы просто разбираем лог Windows Firewall, при помощи регулярного выражения, разбивая его на объекты - один объект на строку. Бог с ней с RAM, не умрем. В данном примере у нас два лога, с двух разных машин. После разбора мы просто сливаем все в один большой массив и гоним по нему поиск, добавляя вершины и ребра графа. Ну и как итог - отображаем его. Вот так, примерно, это выглядит в результате:

![appLinks](https://habrastorage.org/webt/w4/ut/jc/w4utjc6nd9sk07znxb7b0x5yh-0.png)

Надеюсь кому-нибудь пригодится.