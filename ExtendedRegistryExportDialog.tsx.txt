import React, { useCallback, useEffect, useState } from 'react';

import { Button, Col, ElementTypeKind, Grid, IRadioGroupOption, Typography } from '@nbtplatform/inputs';
import { putSecuredData } from '@nbt/api';
import { Configuration } from '@nbt/authentication';

import { useSignalRHub } from '../../hooks';
import {
  NodesBooleanState,
  ActionBooleanState,
  DocumentExportFacadeResponse,
  DocumentGenerationProgressStatus,
  DocumentGenerationProgress,
  ExtendedRegistryExportProps,
  failedExportMessage,
  exportTerminatedMessage,
  startExportProcessMessage,
} from './ExtendedRegistryExportDialog.types';

import { IFacadeServiceQueryDto } from '../BaseRegistry/IBaseRegistryTypes';

import { fileStorageApiSettings } from '../../utilities/settings/uriSettings';

import routes from '../../services/api/routes';

import Dialog from '../Dialog';
import ProgressBarBase from '../ProgressBarBase';

import { base64toBlob } from '../../utilities';
import downloadFile from '../../services/downloadFile';

import { IConfirmDialogDescription } from '../ConfirmDialog/IConfirmDialogTypes';
import { ConfirmDialog, ConfirmDialogButtons } from '../ConfirmDialog';
import RadioGroupInputWrapper from '../input/RadioGroupInput/RadioGroupInputWrapper';
import CustomInput from '../input/CustomInput';
import { IValidationResult, ValidationType } from '../input/CommonInput/IInputValidationTypes';

const ExtendedRegistryExportDialog: React.FC<ExtendedRegistryExportProps> = ({
  closeExportDialog,
  indexName,
  dataRequestParameters,
  columnSettings,
  isGroupedView,
  selectedIds,
  linkedIds,
}) => {
  const [taskId, setTaskId] = useState<string>();
  const [nodeState, setNodeState] = useState<NodesBooleanState>('none');
  const [currentActionState, setCurrentActionState] = useState<ActionBooleanState>('none');
  const [showDictionaryInputForFieldNames, setShowDictionaryInputForFieldNames] = useState(false);
  const [isActionsBlockWhileDocumentGenerated, setIsActionsBlockWhileDocumentGenerated] = useState(false);
  const [dialogOpen, setDialogOpen] = useState(true);
  const [currentSelectedRadioItemKey, setCurrentSelectedRadioItemKey] = useState(1);
  const [boInstanceFields, setBoInstanceFields] = useState<string>();
  const [progressCurrentResult, setProgressCurrentResult] = useState<number>(0);
  const [confirmDialogDescription, setConfirmDialogDescription] = useState<IConfirmDialogDescription>();
  const [validationResultFieldNames, setValidationResultFieldNames] = useState<IValidationResult>();

  const { connection } = useSignalRHub(
    Configuration.Instance.Configuration.apiUrls.reactAppBoInstanceServiceUrl,
    'exportHub',
    undefined,
  );

  useEffect(() => {
    // регистрация обработчиков
    connection?.on('DocumentGenerationProgress', async (model: DocumentGenerationProgress) => {
      const { progress, status, fileTempStorageGuid } = model;

      if (progress > progressCurrentResult) setProgressCurrentResult(progress);

      try {
        // статус документа - сформирован
        if (status === DocumentGenerationProgressStatus.DocumentGenerated && fileTempStorageGuid !== '') {
          // получение файла из временного хранилища
          const downloadFileResponse = await putSecuredData<string>(
            fileStorageApiSettings().downloadFileUri(fileTempStorageGuid),
            {},
          );
          const blob = base64toBlob(downloadFileResponse, '');
          const fileName = 'BoTemplate_' + indexName + '.xlsx';
          // после окончания формирования файла, модальное окно закрывается
          closeCurrentExportDialog();
          // и начинается процесс скачивания файла.
          downloadFile(blob, fileName);
          // снятие блокировки действий
          setIsActionsBlockWhileDocumentGenerated(false);
        }

        // статус документа - остановлено из-за ошибки
        if (status === DocumentGenerationProgressStatus.DocumentGenerationError) {
          throw new Error(failedExportMessage);
        }
      } catch (error: any) {
        setNodeState('openErrorDialog');
        setConfirmDialogDescription({
          title: exportTerminatedMessage,
          message: error.message,
          onConfirm: () => {
            setNodeState('none');
            // снятие блокировки действий
            setIsActionsBlockWhileDocumentGenerated(false);
            // прерывание операции
            setCurrentActionState('isUserCancelExport');
          },
        });
      }
    });

    // если клиент успешно повторно подключается в течение первых четырех попыток(при указанном параметре withAutomaticReconnect),
    // HubConnection вернется в состояние Connected и запустит указанный callback
    connection?.onreconnected(() => {
      if (taskId) connection?.invoke('ExportHubConnection', { taskId });
    });
  }, [connection]);

  // обработка нажатия на кнопку Экспорт - старт операции экспорта
  const handleStartExport = useCallback(async () => {
    // сняты отметки со всех полей
    if (boInstanceFields === '') {
      return;
    }

    // запись состояния - пользователь нажал на кнопку Экспорт
    setCurrentActionState('isProgressStart');
    // блокировка всех действий, кроме "Закрыть"
    setIsActionsBlockWhileDocumentGenerated(true);

    // для выбранных пользователем БО - формирование доп.фильтра
    if (selectedIds && selectedIds.length) {
      (dataRequestParameters as IFacadeServiceQueryDto)?.filter?.push({
        name: 'Id',
        label: 'Уникальный идентификатор',
        fieldType: ElementTypeKind.Number,
        conditionOperatorKind: 0,
        showLinkedCollectionFilter: false,
        value: selectedIds.join(','),
      });
    }

    if (linkedIds) {
      (dataRequestParameters as IFacadeServiceQueryDto)?.filter?.push({
        name: 'Id',
        label: 'Уникальный идентификатор',
        fieldType: ElementTypeKind.Number,
        conditionOperatorKind: 0,
        showLinkedCollectionFilter: false,
        value: linkedIds,
      });
    }

    try {
      // отправка запроса к /IBoInstanceFacadeService/ExportAsync
      const response = await putSecuredData<DocumentExportFacadeResponse>(routes.boInstanceExportUri(), {
        facadeServiceQueryDto: dataRequestParameters,
        boInstanceFields:
          boInstanceFields?.split(',') ??
          columnSettings
            ?.filter((column) => column.isShownInRegistry)
            ?.sort((column1, column2) => (column1.orderNumber >= column2.orderNumber ? 1 : -1))
            ?.map(({ name }) => name),
        ...(isGroupedView ? { isGrouped: true } : {}),
      });
      // запрос выполнен без ошибок
      if (response.status) {
        setTaskId(response.taskId);
        // создание связи клиент SignalR - Идентификатор задачи
        connection?.invoke('ExportHubConnection', { taskId: response.taskId });
      } else {
        throw new Error(failedExportMessage);
      }
    } catch (error: any) {
      setNodeState('openErrorDialog');
      setConfirmDialogDescription({
        title: exportTerminatedMessage,
        message: error.message,
        onConfirm: () => {
          setNodeState('none');
          // снятие блокировки действий
          setIsActionsBlockWhileDocumentGenerated(false);
          // прерывание операции
          setCurrentActionState('isUserCancelExport');
        },
      });
    }
  }, [
    connection,
    JSON.stringify(boInstanceFields),
    JSON.stringify(selectedIds),
    JSON.stringify(dataRequestParameters),
    JSON.stringify(linkedIds),
    JSON.stringify(columnSettings),
  ]);

  // обработка нажатия на кнопку Закрыть - отмена операции экспорта
  const handleStopExport = () => {
    // процесс формирования документа начат
    if (currentActionState === 'isProgressStart') {
      setNodeState('closeButtonClickDialogWhileProgressDocument');
      setConfirmDialogDescription({
        title: 'Внимание',
        message:
          'Идет формирование документа. Если вы закроете форму, скачать результат будет невозможно. Закрыть форму?',
        //  обработка клика на "Да" - закрытие диалог с предупреждением, и формы экспорта
        onConfirm: closeCurrentExportDialog,
        // закрытие диалога с предупреждением
        onCancel: () => {
          setNodeState('none');
        },
      });
    } else {
      closeCurrentExportDialog();
    }
  };

  // закрытие текущего окна экспорта и вызов события closeExportDialog
  const closeCurrentExportDialog = () => {
    connection?.off('DocumentGenerationProgress');
    connection?.stop();
    setDialogOpen(false);
    closeExportDialog();
  };

  // отображение компонента ProgressBar
  const renderProgressBar = () => {
    const computedProgress = Math.floor(progressCurrentResult);
    return <ProgressBarBase id="exportImportProgress" showPercent completed={computedProgress} height={20} />;
  };

  // подготовка кнопок для основного диалогового окна
  const getActionsRight = useCallback(
    () => (
      <>
        <Button variant="outlined" color="danger" onClick={handleStopExport}>
          Закрыть
        </Button>
        <Button color="primary" onClick={handleStartExport} disabled={isActionsBlockWhileDocumentGenerated}>
          Экспорт
        </Button>
      </>
    ),
    [boInstanceFields, isActionsBlockWhileDocumentGenerated, handleStartExport, handleStopExport],
  );

  // отображение диалогового окна с ошибкой
  const renderConfirmDialog = () => {
    if (!confirmDialogDescription) {
      return;
    }
    const dialogParams = {
      open: nodeState === 'openErrorDialog' || nodeState === 'closeButtonClickDialogWhileProgressDocument',
      onConfirm: confirmDialogDescription.onConfirm,
      onCancel: confirmDialogDescription.onCancel,
      title: confirmDialogDescription.title,
      message: confirmDialogDescription.message,
      additionalMessage: confirmDialogDescription.additionalMessage,
      confirmButton:
        nodeState === 'openErrorDialog'
          ? ConfirmDialogButtons.SuccessButtonOk()
          : ConfirmDialogButtons.DangerButton('Да'),
      cancelButton:
        nodeState === 'closeButtonClickDialogWhileProgressDocument' ? ConfirmDialogButtons.SuccessButton('Нет') : {},
    };

    return <ConfirmDialog {...dialogParams} />;
  };

  // обработка изменения выбора переключателя FieldNamesRadioGroupInput
  const handleChangedRadioGroupSelected = (newValue: string | number) => {
    // сброс выбранных полей при переключении элемента RadioGroupSelected
    setBoInstanceFields(undefined);
    setCurrentSelectedRadioItemKey(newValue as number);
    // случай выбора набора столбцов для выгрузки файла
    if (newValue === 2) {
      setShowDictionaryInputForFieldNames(true);
    } else {
      setShowDictionaryInputForFieldNames(false);
    }
  };

  // обработка события изменения столбцов в выпадающем списке
  const handleChange = (newValue: string) => {
    setBoInstanceFields(newValue);
  };

  // подготовка элементов для RadioGroupInput
  const getRadioButtonNamesOptions = () => {
    const names: IRadioGroupOption[] = [
      {
        key: 1,
        value: 'Выгрузить файл с текущим ракурсом реестра',
      },
      {
        key: 2,
        value: 'Изменить набор столбцов для выгрузки файла',
      },
    ];

    return names;
  };

  useEffect(() => {
    if (boInstanceFields === '' && showDictionaryInputForFieldNames) {
      setValidationResultFieldNames({
        validationTypeResult: [{ validationType: ValidationType.Error, validationText: 'Выберите поля для выгрузки' }],
      });
    } else {
      setValidationResultFieldNames(undefined);
    }
  }, [boInstanceFields]);

  return (
    <>
      <Dialog
        open={dialogOpen}
        title="Параметры экспорта реестра"
        maxWidth="sm"
        content={
          <Grid alignItems="center" justifyContent="center">
            <Col wide={12}>
              <RadioGroupInputWrapper
                isDisabled={isActionsBlockWhileDocumentGenerated}
                value={currentSelectedRadioItemKey}
                values={getRadioButtonNamesOptions()}
                id="FieldNamesRadioGroupInput"
                changed={handleChangedRadioGroupSelected}
                readOnly={isGroupedView}
              />
            </Col>
            <Col wide={12}>
              {showDictionaryInputForFieldNames && (
                <CustomInput
                  validationResult={validationResultFieldNames}
                  isDisabled={isActionsBlockWhileDocumentGenerated}
                  id="fieldNamesExportRegistry"
                  elementType={ElementTypeKind.DropDownMultiSelect}
                  changed={handleChange}
                  values={columnSettings?.map((columnSetting) => ({
                    key: columnSetting.name,
                    value: columnSetting.label,
                  }))}
                  value={
                    boInstanceFields ??
                    columnSettings
                      ?.filter((column) => column.isShownInRegistry)
                      ?.sort((column1, column2) => (column1.orderNumber >= column2.orderNumber ? 1 : -1))
                      ?.map(({ name }) => name)
                      ?.join(',')
                  }
                  label="Выберите поля для выгрузки"
                  isRequired
                />
              )}
            </Col>
            {currentActionState === 'isProgressStart' && (
              <Col wide={12}>
                <Typography variant="small">{startExportProcessMessage}</Typography>
              </Col>
            )}
            {currentActionState === 'isProgressStart' && <Col wide={6}>{renderProgressBar()}</Col>}
          </Grid>
        }
        actionsRight={getActionsRight()}
      />
      {renderConfirmDialog()}
    </>
  );
};

export default ExtendedRegistryExportDialog;
