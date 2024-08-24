# Домашнее задание. SQL и реляционные СУБД. Введение в PostgreSQL 

## Вставка блоков кода
``` POWERSHELL
####################################################################################################
<# Author: Ненароков Дмитрий Александрович 
# Created: 2023-07-12
# Modified: 2024-06-25
# Task: https://intraservice.redmond.company/Task/View/343566
# Task2: https://intraservice.redmond.company/Task/View/349937

импорт данных (Статус и дата приема) из xml-выгрузки ЗУП (только в статусе "Принят") в аттрибут ExtensionAttribute4 пользователя

"Принят: ГГГГ-ММ-ДД" ===>  user.ExtensionAttribute4 (убрал и перенес в отдельный скрипт так как были баги с синхронизацией)
"HRManagerName: ФИО ответсвенного за прием менеджера HR" ==> user.ExtensionAttribute7
"HRManagerEmail: e-mail ответсвенного за прием менеджера HR" ==> user.ExtensionAttribute8
 user.ExtensionAttribute5 - резервируем для отметок об отправке адаптационных писем 1m 2m 3m
#>
####################################################################################################

#Импортируем данные из xml-выгрузки ЗУП
[xml]$userfile = Get-Content -Path "\\tps.local\it\Scripts\1C-to-ADDS-GUID\company.xml"
#Выбираем значения xml-нод "Контакт" в которых Значение "UserID" заполнено
$xml_usrs = $userfile.selectNodes("//Контакт//Тип") | Where-Object {$_.InnerXML -eq 'UserID'} 
$Departments = $userfile.SelectNodes("//КоммерческаяИнформация//Классификатор//Подразделения//Подразделение") #| Where-Object {$_.InnerXML -eq 'Ид'} 

$Path = "C:\Scripts\HR-Adaptation-notify"
$Date=(Get-Date -Format "yyyy/MM/dd").ToString()
# Путь к файлу логов
$log="$Path\log\sync-$Date.log"
# Функция для логирования действий скрипта
function to_log_short ([string]$msg=" "){
    [string]$message=(Get-Date -UFormat "%H:%M:%S").ToString()+" ** "+"$msg"#+"`n"
    $message | Out-File $log -Encoding utf8 -append
} 

# Очищаем аттрибутs Extensionattribute7,8 для пользователей созданных за последние 3 дня чтобы при создании сотрудниками ГО пользователя копированием не было влияния на рассылку
$now = get-date 
$date = $now.AddDays(-3)
$lastcreatedusers = get-aduser -SearchBase 'OU=Offices,DC=tps,DC=local' -filter 'whencreated -ge $date' -properties Displayname, Name, Whencreated, extensionattribute4, extensionattribute5, extensionattribute7, extensionattribute8
{
$Displayname =  $lastcreateduser.DisplayName
$lastcreateduser | Set-ADUser -Clear @("extensionAttribute7","extensionAttribute8") -verbose
Write-Host "стираем аттрибуты extensionattribute7,8 для пользователя $Displayname" -ForegroundColor Green
to_log_short "стираем аттрибут extensionattribute7,8 для пользователя $Displayname"
}

# Обнуляем значения переменных типа System.Array чтобы при втором проходе в ISE массив не рос 
$users =@()
$usersummary =@()
$prinyat = @()
$login = $null
# статусы сотрудников для выборки
$needed_statuses='Принят' #,'Перемещен'#, 'Уволен'


Write-Host "Генерируем итоговый массив данных по сотрудникам в статусе Принят из xml-выгрузки ЗУП..." -ForegroundColor Cyan
$counter=0

# Для всех нод "Контакт" в исходном xml-файле ЦИКЛ
foreach($xml_usr in $xml_usrs) 

    {   
        $counter++
        Write-Progress -Activity 'Processing employes array' -CurrentOperation $($xml_usr.ParentNode.ParentNode.ParentNode.Наименование) -PercentComplete (($counter / $xml_usrs.count) * 100)
        
        $HRManagerGUID = $null
        $HR = @()
        $HRManagerGUID = $xml_usr.ParentNode.ParentNode.ParentNode.ОтветственныйЗаПрием
        $HR =  ($xml_usrs.ParentNode.ParentNode.Parentnode | where {$_.Ид -eq "$HRManagerGUID" -and $_.Ид -ne '00000000-0000-0000-0000-000000000000' -and $_.Ид -ne 'null'})
        $HRManagerEmail = ($HR.ChildNodes.ChildNodes | where {$_.Тип -eq 'Почта'}).Значение
        $HRManagerName = $HR.Наименование  
               
        $users = [PSCustomObject]@{
        login = ($xml_usr.ParentNode.Значение).Trim();
        Username  = ($xml_usr.ParentNode.ParentNode.ParentNode.Наименование).Trim();
        State  = ($xml_usr.ParentNode.ParentNode.ParentNode.ИсторияСостояний.Состояние.Значение).Trim();
        #StateChangeDate = ($xml_usr.ParentNode.ParentNode.ParentNode.ИсторияСостояний.Состояние.ДатаИзменения).Trim();
        GUID = ($xml_usr.ParentNode.ParentNode.ParentNode.Ид).Trim();
        HRManangerGUID = ($HRManagerGUID).Trim();
        HRManangerEmail = $HRManagerEmail;
        HRManagerName = $HRManagerName;
        
                  } #[PSCustomObject]@

        [array]$usersummary += $users # прибавляем к массиву элемент на каждом шаге цикла
}#foreach КОНЕЦ ЦИКЛА
Write-Host "ГОТОВО !!!" -ForegroundColor Green


$prinyat_employes = $usersummary | Where-Object {$_.State -in $needed_statuses} 


$usercounter = 0 
# Для всех элементов массива $prinyat_employes  ЦИКЛ
foreach ($prinyat_employe in $prinyat_employes)
{
$usercounter++
Write-Progress -Activity 'Sync userStateChangeDate to ExtensionAttribute4 ' -CurrentOperation $prinyat_employe.Username -PercentComplete (($usercounter/ $prinyat_employes.count) * 100)
$login = $prinyat_employe.Login

    try{
         $aduser = get-aduser  –Identity $login -Properties Name, ExtensionAttribute4, ExtensionAttribute7, ExtensionAttribute8, Samaccountname | Select-Object Name, ExtensionAttribute4, ExtensionAttribute7, ExtensionAttribute8, Samaccountname
         $aduserlogin = $aduser.Samaccountname
        }
        catch {
            to_log_short("ZUP login not found in AD: $login")
            continue;
            }
 
$employename = $prinyat_employe.Username

$HRManName = $prinyat_employe.HRManagerName
$HRManEmail = $prinyat_employe.HRManangerEmail

#[string]$prinyat = "Принят: $userStateChangeDate"
[string]$HRNameAttribute = "HRManagerName: $HRManName"
[string]$HREmailAttribute = "HRManagerEmail: $HRManEmail"


  if  ($aduser.ExtensionAttribute7 -eq $null)
  {
  Write-Host "Заполняем ExtensionAttribute7 = $HRNameAttribute для XML: $employename Login: $aduserlogin" -ForegroundColor Green
  to_log_short "Заполняем ExtensionAttribute7 = $HRNameAttribute для XML: $employename Login: $aduserlogin"
    try {
    
    Set-ADUser -Identity $aduserlogin -Replace @{ExtensionAttribute7="$HRNameAttribute";}
   
                                                      
        }
    catch {
           Write-Host "Ошибка записи аттрибута AD для $aduserlogin" -ForegroundColor Red
           to_log_short "Ошибка записи аттрибута AD для $aduserlogin"
        }
    
  }
else
  {
   Write-Host "$employename * ExtensionAttribute7 уже заполнен !!!" -ForegroundColor   DarkCyan
   to_log_short "$employename * ExtensionAttribute7 уже заполнен !!!"
  }

  if  ($aduser.ExtensionAttribute8 -eq $null)
  {
  Write-Host "Заполняем ExtensionAttribute8 = $HREmailAttribute для XML: $employename Login: $aduserlogin" -ForegroundColor Green
  to_log_short "Заполняем ExtensionAttribute8 = $HREmailAttribute для XML: $employename Login: $aduserlogin"
    try {
    
    
    Set-ADUser -Identity $aduserlogin -Replace @{ExtensionAttribute8="$HREmailAttribute";}
                                                      
        }
    catch {
           Write-Host "Ошибка записи аттрибута AD для $aduserlogin" -ForegroundColor Red
           to_log_short "Ошибка записи аттрибута AD для $aduserlogin"
        }
    
  }
else
  {
   Write-Host "$employename * ExtensionAttribute8 уже заполнен !!!" -ForegroundColor   DarkCyan
   to_log_short "$employename * ExtensionAttribute8 уже заполнен !!!"
  }

  } #foreach КОНЕЦ ЦИКЛА

```
