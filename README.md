-- SELECT * FROM Boxing WHERE FakeSerialNumber = 'bxdc19622062'

DECLARE @BoxingID dbo.BIDENT = 101101169  --ID короба
DECLARE @TransferID dbo.BIDENT = (SELECT TransferID FROM Boxing WHERE ID = @BoxingID)

--Проверяем короб на финальный статус
IF EXISTS (SELECT 1 FROM Boxing WHERE ID = @BoxingID AND BoxingStatusID = 14)
BEGIN
    SELECT 'Короб в финальном статусе'
    RETURN
END

--Проверяем перемещение на тип маршрута Пополение с РЦ
IF EXISTS (SELECT 1
           FROM Transfer t
           INNER JOIN TransferType tt ON t.TransferTypeID=tt.ID
           WHERE t.ID = @TransferID AND tt.TransferTypeGroupID != 12)
BEGIN
    SELECT 'У перемещения тип маршрута не Пополнение с РЦ'
    RETURN
END

--проверка на последний короб в перемещении
IF NOT EXISTS (SELECT 1 FROM Boxing WHERE TransferID = @TransferID AND ID != @BoxingID AND BoxingStatusID != 14)
BEGIN

    -- Начитываем во временную таблицу все задания короба
    DROP TABLE IF EXISTS #StoreTasks
    SELECT st.*
    INTO #StoreTasks
    FROM StoreTask st
    INNER JOIN StoreTaskBoxing stb ON st.ID=stb.StoreTaskID
    INNER JOIN StoreTaskStatus sts ON st.StoreTaskStatusID=sts.ID
    WHERE stb.BoxingID = @BoxingID
    AND sts.FinishStatus != 1

    SELECT 'Перемещение' [Перемещение], * FROM Transfer WHERE ID = @TransferID
    SELECT 'Короб' [Короб], * FROM Boxing WHERE ID = @BoxingID
    SELECT 'Задания у короба' [Задания у короба], * FROM StoreTask WHERE ID = ANY (SELECT ID FROM #StoreTasks)
    SELECT 'Накладные от заданий' [Накладные от заданий],* FROM StoreTaskConsig WHERE StoreTaskID = ANY (SELECT ID FROM #StoreTasks)
    SELECT  TOP(1) 'Накладные экземпляра' [Накладные экземпляра], c.*
    FROM Exemplar E
    left join ConsigItemCellExemplar CICE ON CICE.ExemplarID=E.ID
    left join ConsigItemCell CIC ON CIC.ID=CICE.ConsigItemCellID
    left join ConsigItem CI ON CI.ID=CIC.ConsigItemID
    left join Consig C ON CI.ConsigID=C.ID
    INNER JOIN ExemplarInBoxing eib ON e.ID=eib.ExemplarID
    WHERE eib.BoxingID = @BoxingID

    --Открываем курсор для завершения заданий короба
    DECLARE @StoreTaskID BIDENT
    declare CE cursor local static for
        SELECT ID FROM #StoreTasks

    open CE

    fetch next from CE into @StoreTaskID

    while @@fetch_status = 0
    begin

        EXEC StoreTaskFinish @StoreTaskID

        fetch next from CE into @StoreTaskID

    end

    close CE
    deallocate CE
    --Закрываем курсор для завершения заданий короба

    -- Начитываем не завершенные задания 49 типа по перемещению во временную таблицу
    DROP TABLE IF EXISTS #StoreTask49
    SELECT st.*
    INTO #StoreTask49
    FROM StoreTask st
    INNER JOIN TransferStoreTask tst ON st.ID=tst.StoreTaskID
    WHERE st.StoreTaskStatusID != 209
    AND st.StoreTaskTypeID = 49
    AND tst.TransferID = @TransferID

    --Если есть не завершенное задание 49 типа, переводим в конечный статус 209
    IF EXISTS ( SELECT 1 FROM #StoreTask49 )
    BEGIN
        update dbo.StoreTask
        set StoreTaskStatusID = 209
        where ID = ANY (SELECT ID FROM #StoreTask49)
    END

        IF EXISTS ( SELECT 1 FROM Transfer
                    WHERE ID = @TransferID
                    AND TransferStatusID = 51)
        BEGIN
            EXEC ConsigStatusDispatcher @TransferID, 52
        END

    --Переводим короб в конечный статус
    EXEC [dbo].[BoxingFinishProcess] @BoxingID

    SELECT 'Перемещение' [Перемещение], * FROM Transfer WHERE ID = @TransferID
    SELECT 'Короб' [Короб], * FROM Boxing WHERE ID = @BoxingID
    SELECT 'Задания у короба' [Задания у короба], * FROM StoreTask WHERE ID = ANY (SELECT ID FROM #StoreTasks)
    SELECT  TOP(1) 'Накладные экземпляра' [Накладные экземпляра], c.*
    FROM Exemplar E
    left join ConsigItemCellExemplar CICE ON CICE.ExemplarID=E.ID
    left join ConsigItemCell CIC ON CIC.ID=CICE.ConsigItemCellID
    left join ConsigItem CI ON CI.ID=CIC.ConsigItemID
    left join Consig C ON CI.ConsigID=C.ID
    INNER JOIN ExemplarInBoxing eib ON e.ID=eib.ExemplarID
    WHERE eib.BoxingID = @BoxingID
    SELECT 'Обработка короба завершена.'

END
ELSE
BEGIN
    SELECT REPLACE(REPLACE('Указанный короб ID = {1} не является последним в перемещении №{0}', '{0}', @TransferID ), '{1}', @BoxingID)
END