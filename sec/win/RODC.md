**Read only domain controller**

# Назначение и особенности эксплуатации

По задумки мелкомягких предназначен для аутентификации ограниченной группы пользователей, например сотрудников филиала, поэтому идея выностить его в DMZ и использовать для аутентификации всех имеющихся пользователей, включая администратора домена, имеет место быть, но не соответствует его прямому назначению.

## Учетные записи, обладающие административным доступом к RODC

RODC не входит в Tier Zero, поэтому на нем не должно быть ничего, что позволит получить доступ к ресурсам из Tier Zero (но защищать объект компьютера RODC производитель советует как объект Tier Zero). На контроллере домена, как известно, нет локальных учетных записей и групп безопасности, роль локального администратора выполныет администратор домена. Но для RODC, в силу упомянутого выше ограничения, такой механизм не подходит, поэтому для него было сделано исключение - администраторами RODC являются перечисленные в поле ___managedBy___ пользователи и группы. 
Отсюда вывод - внимательно следите, кто имеет право на изменения свойства ___managedBy___ у объекта компьютера RODC и сами не пихайте туда лишнего.

##Учетные записи, которые может аутентифицировать RODC
Собственно, сабж содержиться в атрибуте ___msDS-RevealOnDemandGroup___ объекта компьютера RODC.
А те, кого он аутенифицировать не должен, перечисляются в атрибуте ___msDS-NeverRevealGroup___.
Логично, что если RODC может кого-то аутенифицировать, то на нем появляется хэш его пароля, который могут похитить злобные хацкеры, поэтому имеет смысл закатать в msDS-NeverRevealGroup все учетные записи, наделенные  мало-мальскими правами.
Успешно прошедшие аутенификацию пользователи перечисляются в атрибуте ___msDS-AuthenticatedToAccountList___.
И еще до кучи: ___msDS-Revealed___
Это целая группа из нескольких атрибутов:
  1. ___msDS-RevealedUsers___ — список объектов, учетные данные которых когда‑либо кешировались на RODC;
  2. ___msDS-RevealedDSAs___ — список RODC, которые кешировали пароль пользователя;
  3. ___msDS-RevealedList___ — список объектов, учетные данные которых были успешно сохранены на RODC.

## После аутентификации
Как только RODC успешно аутентифицировал пользователя домена, он выпускает ему TGT-билетик ~на трамвай~. Но вспоминаем, что RODC находиться за пределами Tier Zero, поэтому пароль от учетной записи ___KBRTGT___ (которым шифруются билеты kerberos) ему не светит, поэтому просто так выпустить TGT он не сможет. Для  этого, при создании RODC, создается дополнительная учетная запись KRBTGT_XXXXX, число в окончании имени которой является идентификатором специальных kerberos-билетов, которые обязательны к приему на полноценных DC. Это же число сохраняется в атрибуте ___msDS-SecondaryKrbTgtNumber___ объекта новой учетной записи. А имя этой учетной записи сохраняется в атрибуте ___msDS-KrbTgtLink___ объекта компьютера RODC. И обратная связь тоже сохраняется - в атрибуте ___msDS-KrbTgtLinkBl__ (от слова backlink) объекта учётной записи содержится имя RODC'а. А самое главное в том, что аккаунт компьютера RODC'а имеет право на смену пароля учетной записи KRBTGT_XXXXX. 
Итого, RODC может выпускать TGT, которые используются для запроса TGS-битлетов, позволяющий получить доступ к ресурсам домена. Но TGS-битлеты выпускает уже полноценный KDC, расположенный на контроллере домена и ему эти TGT со специальными идентификаторами не указ. Поэтому, DC, получив от RODC TGS-запрос, подписанный номерным билетиком проверяет, входит ли аккаунт, для которого запрашивают тикет, в список пользователей, которых может атентифицировать RODC (___msDS-RevealOnDemandGroup___) и отсутствует ли он в списке пользователей, которых тот аутентифицировать не должен (___msDS-NeverRevealGroup___). Есл все в порядке, то DC передобписывает запрос нормальным TGT и передает в KDC.
 
