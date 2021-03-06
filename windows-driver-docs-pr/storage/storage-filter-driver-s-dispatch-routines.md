---
title: Storage Filter Driver's Dispatch Routines
description: Storage Filter Driver's Dispatch Routines
ms.assetid: 0d1af035-537f-4632-800b-eb344dc5a3c8
keywords:
- storage filter drivers WDK , dispatch routines
- filter drivers WDK storage , dispatch routines
- SFD WDK storage , dispatch routines
- dispatch routines WDK storage
ms.date: 04/20/2017
ms.localizationpriority: medium
---

# Storage Filter Driver's Dispatch Routines


## <span id="ddk_storage_filter_drivers_dispatch_routines_kg"></span><span id="DDK_STORAGE_FILTER_DRIVERS_DISPATCH_ROUTINES_KG"></span>


Like any other higher-level kernel-mode driver, a storage filter driver (SFD) must have one or more *Dispatch* routines to handle every IRP\_MJ\_XXX request for which the underlying storage driver supplies a *Dispatch* entry point. Depending on the nature of its device, the *Dispatch* entry point of an SFD might do one of the following for any given request:

-   For a request that requires no special handling, set up the I/O stack location in the IRP for the next-lower driver, possibly call [**IoSetCompletionRoutine**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iosetcompletionroutine) to set up its [*IoCompletion*](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nc-wdm-io_completion_routine) routine for the IRP, and pass the IRP on for further processing by lower drivers with [**IoCallDriver**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iocalldriver).

-   For a request already handled by a storage class driver, modify the SRB in the I/O stack location of the IRP before setting up the I/O stack location, possibly set an [*IoCompletion*](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nc-wdm-io_completion_routine) routine, and pass the IRP to the next-lower driver with [**IoCallDriver**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iocalldriver).

-   Set up a new IRP with an SRB and CDB for its device, call [**IoSetCompletionRoutine**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iosetcompletionroutine) so the SRB (and the IRP if the driver calls [**IoAllocateIrp**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-ioallocateirp) or [**IoBuildAsynchronousFsdRequest**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iobuildasynchronousfsdrequest)) can be freed, and pass the IRP on with [**IoCallDriver**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iocalldriver)

    An SFD is most likely to set up new IRPs with the major function code [**IRP\_MJ\_INTERNAL\_DEVICE\_CONTROL**](https://docs.microsoft.com/windows-hardware/drivers/kernel/irp-mj-internal-device-control).

## <span id="Processing_requests"></span><span id="processing_requests"></span><span id="PROCESSING_REQUESTS"></span>Processing requests


For requests that require no special handling, the *Dispatch* routine of an SFD usually calls [**IoSkipCurrentIrpStackLocation**](https://docs.microsoft.com/windows-hardware/drivers/kernel/mm-bad-pointer) with an input IRP and then calls [**IoCallDriver**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iocalldriver) with pointers to the class driver's device object and the IRP. Note that an SFD seldom sets its [*IoCompletion*](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nc-wdm-io_completion_routine) routine in IRPs that require no special handling both because a call to the *IoCompletion* routine is unnecessary and because it degrades I/O throughput for the driver's devices. If an SFD does set an *IoCompletion* routine, it calls [**IoCopyCurrentIrpStackLocationToNext**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iocopycurrentirpstacklocationtonext) instead of **IoSkipCurrentIrpStackLocation** and then calls [**IoSetCompletionRoutine**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iosetcompletionroutine) before calling **IoCallDriver**.

For requests that do require special handling, the SFD can do the following:

1.  Create a new IRP with [**IoBuildDeviceIoControlRequest**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iobuilddeviceiocontrolrequest), [**IoAllocateIrp**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-ioallocateirp), [**IoBuildSynchronousFsdRequest**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iobuildsynchronousfsdrequest), or [**IoBuildAsynchronousFsdRequest**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iobuildasynchronousfsdrequest), usually specifying an I/O stack location for itself.

2.  Check the returned IRP pointer for **NULL** and return **STATUS\_INSUFFICIENT\_RESOURCES** if an IRP could not be allocated.

3.  If the driver-created IRP includes an I/O stack location for the SFD, call [**IoSetNextIrpStackLocation**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iosetnextirpstacklocation) to set up the IRP stack location pointer. Then, call [**IoGetCurrentIrpStackLocation**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iogetcurrentirpstacklocation) to get a pointer to its own I/O stack location in the driver-created IRP and set up it up with state to be used by its own [*IoCompletion*](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nc-wdm-io_completion_routine) routine.

4.  Call [**IoGetNextIrpStackLocation**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iogetnextirpstacklocation) to get a pointer to the next-lower driver's I/O stack location in the driver-created IRP and set it up with the major function code **IRP\_MJ\_SCSI** and an SRB (see [Storage Class Drivers](storage-class-drivers.md)).

5.  Translate data to be transferred to the device into a device-specific, nonstandard format if necessary.

6.  Call [**IoSetCompletionRoutine**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iosetcompletionroutine) if the driver allocated any memory, such as memory for an SRB, SCSI request-sense buffer, MDL, and/or IRP with a call to [**IoAllocateIrp**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-ioallocateirp) or [**IoBuildAsynchronousFsdRequest**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iobuildasynchronousfsdrequest), or if the driver must translate data transferred from the device in a device-specific, nonstandard format.

7.  Pass the driver-created IRP to (and through) the next-lower driver with [**IoCallDriver**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdm/nf-wdm-iocalldriver).

## <span id="Handling_SRB_formats"></span><span id="handling_srb_formats"></span><span id="HANDLING_SRB_FORMATS"></span>Handling SRB formats


Starting with Windows 8, an SFD filtering between the class driver and the port driver must check for the supported SRB format. Specifically, this involves detecting the SRB format and accessing the members of the structure correctly. The SRB in the IRP is either an [**SCSI\_REQUEST\_BLOCK**](https://docs.microsoft.com/windows-hardware/drivers/ddi/srb/ns-srb-_scsi_request_block) and or an [**STORAGE\_REQUEST\_BLOCK**](https://docs.microsoft.com/windows-hardware/drivers/ddi/srb/ns-srb-_storage_request_block). A filter driver can determine ahead of time which SRBs are supported by the port driver below by issuing an [**IOCTL\_STORAGE\_QUERY\_PROPERTY**](https://docs.microsoft.com/windows-hardware/drivers/ddi/ntddstor/ni-ntddstor-ioctl_storage_query_property) request and specifying the **StorageAdapterProperty** identifier. The **SrbType** and **AddressType** values returned in the [**STORAGE\_ADAPTER\_DESCRIPTOR**](https://docs.microsoft.com/windows-hardware/drivers/ddi/ntddstor/ns-ntddstor-_storage_adapter_descriptor) structure indicate the SRB format and addressing scheme used by the port driver. Any new SRBs allocated and sent by the filter driver must be of the type returned by the query.

Similarly, starting with Windows 8, SFDs supporting only SRBs of the [**SCSI\_REQUEST\_BLOCK**](https://docs.microsoft.com/windows-hardware/drivers/ddi/srb/ns-srb-_scsi_request_block) type must check that the **SrbType** value returned in the [**STORAGE\_ADAPTER\_DESCRIPTOR**](https://docs.microsoft.com/windows-hardware/drivers/ddi/ntddstor/ns-ntddstor-_storage_adapter_descriptor) structure is set to **SRB\_TYPE\_SCSI\_REQUEST\_BLOCK**. To handle the situation when **SrbType** is set to **SRB\_TYPE\_STORAGE\_REQUEST\_BLOCK** instead, the filter driver must set a completion routine for [**IOCTL\_STORAGE\_QUERY\_PROPERTY**](https://docs.microsoft.com/windows-hardware/drivers/ddi/ntddstor/ni-ntddstor-ioctl_storage_query_property) when the **StorageAdapterProperty** identifier is set in the request sent by drivers above it. In the completion routine, the **SrbType** member in the **STORAGE\_ADAPTER\_DESCRIPTOR** is modified to **SRB\_TYPE\_SCSI\_REQUEST\_BLOCK** to correctly set the supported type.

The following is an example of a filter dispatch routine which handles both SRB formats.

```ManagedCPlusPlus
NTSTATUS FilterScsiIrp(
    PDEVICE_OBJECT DeviceObject,
    PIRP Irp
    )
{
    PFILTER_DEVICE_EXTENSION  deviceExtension = DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION irpStack = IoGetCurrentIrpStackLocation(Irp);
    NTSTATUS status;
    PSCSI_REQUEST_BLOCK srb;   
    ULONG srbFunction;
    ULONG srbFlags;

    //
    // Acquire the remove lock so that device will not be removed while
    // processing this irp.
    //

    status = IoAcquireRemoveLock(&deviceExtension->RemoveLock, Irp);

    if (!NT_SUCCESS(status)) {
        Irp->IoStatus.Status = status;
        IoCompleteRequest(Irp, IO_NO_INCREMENT);
        return status;
    }

    srb = irpStack->Parameters.Scsi.Srb;
     
    if (srb->Function == SRB_FUNCTION_STORAGE_REQUEST_BLOCK) {
        srbFunction = ((PSTORAGE_REQUEST_BLOCK)srb)->SrbFunction;
        srbFlags = ((PSTORAGE_REQUEST_BLOCK)srb)->SrbFlags;
    } else {
        srbFunction = srb->Function;
        srbFlags = srb->SrbFlags;
    }

    if (srbFunction == SRB_FUNCTION_EXECUTE_SCSI) {
        if (srbFlags & SRB_FLAGS_UNSPECIFIED_DIRECTION) {
            // ...

            // filter processing for SRB_FUNCTION_EXECUTE_SCSI

            // ...
        }
    }

    IoMarkIrpPending(Irp);
    IoCopyCurrentIrpStackLocationToNext(Irp);
    IoSetCompletionRoutine(Irp,
                           FilterScsiIrpCompletion,
                           DeviceExtension->DeviceObject,
                           TRUE, TRUE, TRUE);
    IoCallDriver(DeviceExtension->TargetDeviceObject, Irp);

    return STATUS_PENDING; 
}
```

## <span id="Setting_up_requests"></span><span id="setting_up_requests"></span><span id="SETTING_UP_REQUESTS"></span>Setting up requests


Like a storage class driver, an SFD might have *BuildRequest* or *SplitTransferRequest* routines to be called from the driver's *Dispatch* routines, or might implement the same functionality inline.

For more information about *BuildRequest* and *SplitTransferRequest* routines, see [Storage Class Drivers](storage-class-drivers.md). For more information about general requirements for *Dispatch* routines, see [Writing Dispatch Routines](https://docs.microsoft.com/windows-hardware/drivers/kernel/writing-dispatch-routines).

 

 




