# dashboard-test

1. Сборка
npm install - установить зависимости
npm run serve - запустить сервер разработки

2. Описание
холст:
- размер 5000х5000, центруется по умолчанию,
- можно перемещать внутри вьюпорта

блоки:
- можно перемещать в любое время
- минимальное расстояние между блоками - 30
- при обнаружении наложения: новый блок удаляется, старый встает на свое прежнее место

связи:
- разрешена двухсторонняя связь
- запрещено дублирование