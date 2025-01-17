---
ms.assetid: dd2a1e01-c284-4d62-963e-f59f58dca61a
description: This article describes how to import media from a device, including searching for available media sources, importing files such as photos and sidecar files, and deleting the imported files from the source device.
title: Import media
ms.date: 02/08/2017
ms.topic: article
keywords: windows 10, uwp
ms.localizationpriority: medium
---
# Import media from a device

This article describes how to import media from a device, including searching for available media sources, importing files such as videos, photos, and sidecar files, and deleting the imported files from the source device.

> [!NOTE] 
> The code in this article was adapted from the [**MediaImport UWP app sample**](https://github.com/Microsoft/Windows-universal-samples/tree/master/Samples/MediaImport) . You can clone or download this sample from the [**Universal Windows app samples Git repo**](https://github.com/Microsoft/Windows-universal-samples) to see the code in context or to use it as a starting point for your own app.

## Create a simple media import UI
The example in this article uses a minimal UI to enable the core media import scenarios. To see how to create a more robust UI for a media import app, see the [**MediaImport sample**](https://github.com/Microsoft/Windows-universal-samples/tree/master/Samples/MediaImport). The following XAML creates a stack panel with the following controls:
* A [**Button**](https://docs.microsoft.com/uwp/api/Windows.UI.Xaml.Controls.Button) to initiate searching for sources from which media can be imported.
* A [**ComboBox**](https://docs.microsoft.com/uwp/api/Windows.UI.Xaml.Controls.ComboBox) to list and select from the media import sources that are found.
* A [**ListView**](https://docs.microsoft.com/uwp/api/Windows.UI.Xaml.Controls.ListView) control to display and select from the media items from the selected import source.
* A **Button** to initiate importing media items from the selected source.
* A **Button** to initiate deleting the items that have been imported from the selected source.
* A **Button** to cancel an asynchronous media import operation.

[!code-xml[ImportXAML](./code/PhotoImport_Win10/cs/MainPage.xaml#SnippetImportXAML)]

## Set up your code-behind file
Add *using* directives to include the namespaces used by this example that are not already included in the default project template.

[!code-cs[Using](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetUsing)]

## Set up task cancellation for media import operations

Because media import operations can take a long time, they are performed asynchronously using [**IAsyncOperationWithProgress**](https://docs.microsoft.com/uwp/api/Windows.Foundation.IAsyncOperationWithProgress_TResult_TProgress_). Declare a class member variable of type [**CancellationTokenSource**](https://docs.microsoft.com/dotnet/api/system.threading.cancellationtokensource?redirectedfrom=MSDN) that will be used to cancel an in-progress operation if the user clicks the cancel button.

[!code-cs[DeclareCts](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetDeclareCts)]

Implement a handler for the cancel button. The examples shown later in this article will initialize the **CancellationTokenSource** when an operation begins and set it to null when the operation completes. In the cancel button handler, check to see if the token is null, and if not, call [**Cancel**](https://docs.microsoft.com/dotnet/api/system.threading.cancellationtokensource.cancel?redirectedfrom=MSDN#System_Threading_CancellationTokenSource_Cancel) to cancel the operation.

[!code-cs[OnCancel](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetOnCancel)]

## Data binding helper classes

In a typical media import scenario you show the user a list of available media items to import, there can be a large number of media files to choose from and, typically, you want to show a thumbnail for each media item. For this reason, this example uses three helper classes to incrementally load entries into the ListView control as the user scrolls down through the list.

* **IncrementalLoadingBase** class - Implements the [**IList**](https://docs.microsoft.com/dotnet/api/system.collections.ilist?redirectedfrom=MSDN), [**ISupportIncrementalLoading**](https://docs.microsoft.com/uwp/api/windows.ui.xaml.data.isupportincrementalloading), and [**INotifyCollectionChanged**](https://msdn.microsoft.com/library/windows/apps/system.collections.specialized.inotifycollectionchanged(v=vs.105).aspx) to provide the base incremental loading behavior.
* **GeneratorIncrementalLoadingClass** class - Provides an implementation of the incremental loading base class.
* **ImportableItemWrapper** class - A thin wrapper around the [**PhotoImportItem**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportItem) class to add a bindable [**BitmapImage**](https://docs.microsoft.com/uwp/api/Windows.UI.Xaml.Media.Imaging.BitmapImage) property for the thumbnail image for each imported item.

These classes are provided in the [**MediaImport sample**](https://github.com/Microsoft/Windows-universal-samples/tree/master/Samples/MediaImport) and can be added to your project without modifications. After adding the helper classes to your project, declare a class member variable of type **GeneratorIncrementalLoadingClass** that will be used later in this example.

[!code-cs[GeneratorIncrementalLoadingClass](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetGeneratorIncrementalLoadingClass)]


## Find available sources from which media can be imported

In the click handler for the find sources button, call the static method [**PhotoImportManager.FindAllSourcesAsync**](https://docs.microsoft.com/uwp/api/windows.media.import.photoimportmanager.findallsourcesasync) to start the system searching for devices from which media can be imported. After awaiting the completion of the operation, loop through each [**PhotoImportSource**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportSource) object in the returned list and add an entry to the **ComboBox**, setting the **Tag** property to the source object itself so it can be easily retrieved when the user makes a selection.

[!code-cs[FindSourcesClick](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetFindSourcesClick)]

Declare a class member variable to store the user's selected import source.

[!code-cs[DeclareImportSource](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetDeclareImportSource)]

In the [**SelectionChanged**](https://docs.microsoft.com/uwp/api/windows.ui.xaml.controls.primitives.selector.selectionchanged) handler for the import source **ComboBox**, set the class member variable to the selected source and then call the **FindItems** helper method which will be shown later in this article. 

[!code-cs[SourcesSelectionChanged](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetSourcesSelectionChanged)]

## Find items to import

Add class member variables of type [**PhotoImportSession**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportSession) and [**PhotoImportFindItemsResult**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportFindItemsResult) which will be used in the following steps.

[!code-cs[DeclareImport](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetDeclareImport)]

In the **FindItems** method, initialize the **CancellationTokenSource** variable so it can be used to cancel the find operation if necessary. Within a **try** block, create a new import session by calling [**CreateImportSession**](https://docs.microsoft.com/uwp/api/windows.media.import.photoimportsource.createimportsession) on the [**PhotoImportSource**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportSource) object selected by the user. Create a new [**Progress**](https://docs.microsoft.com/dotnet/api/system.progress-1?redirectedfrom=MSDN) object to provide a callback to display the progress of the find operation. Next, call **[FindItemsAsync](https://docs.microsoft.com/uwp/api/windows.media.import.photoimportsession.finditemsasync)** to start the find operation. Provide a [**PhotoImportContentTypeFilter**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportContentTypeFilter) value to specify whether photos, videos, or both should be returned. Provide a [**PhotoImportItemSelectionMode**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportItemSelectionMode) value to specify whether all, none, or only the new media items are returned with their [**IsSelected**](https://docs.microsoft.com/uwp/api/windows.media.import.photoimportitem.isselected) property set to true. This property is bound to a checkbox for each media item in our ListBox item template.

**FindItemsAsync** returns an [**IAsyncOperationWithProgress**](https://docs.microsoft.com/uwp/api/Windows.Foundation.IAsyncOperationWithProgress_TResult_TProgress_). The extension method [**AsTask**](https://docs.microsoft.com/dotnet/api/system?redirectedfrom=MSDN) is used to create a task that can be awaited, can be cancelled with the cancellation token, and that reports progress using the supplied **Progress** object.

Next the data binding helper class, **GeneratorIncrementalLoadingClass** is initialized. **FindItemsAsync**, when it returns from being awaited, returns a [**PhotoImportFindItemsResult**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportFindItemsResult) object. This object contains status information about the find operation, including the success of the operation and the count of different types of media items that were found. The [**FoundItems**](https://docs.microsoft.com/uwp/api/windows.media.import.photoimportfinditemsresult.founditems) property contains a list of [**PhotoImportItem**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportItem) objects representing the found media items. The **GeneratorIncrementalLoadingClass** constructor takes as arguments the total count of items that will be loaded incrementally, and a function that generates new items to be loaded as needed. In this case, the provided lambda expression creates a new instance of the **ImportableItemWrapper** which wraps **PhotoImportItem** and includes a thumbnail for each item. Once the incremental loading class has been initialized, set it to the [**ItemsSource**](https://docs.microsoft.com/uwp/api/windows.ui.xaml.controls.itemscontrol.itemssource) property of the **ListView** control in the UI. Now, the found media items will be loaded incrementally and displayed in the list.

Next, the status information of the find operation are output. A typical app will display this information to the user in the UI, but this example simply outputs the information to the debug console. Finally, set the cancellation token to null because the operation is complete.

[!code-cs[FindItems](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetFindItems)]

## Import media items

Before implementing the import operation, declare a [**PhotoImportImportItemsResult**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportImportItemsResult) object to store the results of the import operation. This will be used later to delete media items that were successfully imported from the source.

[!code-cs[DeclareImportResult](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetDeclareImportResult)]

Before starting the media import operation initialize the **CancellationTokenSource** variable and by setting the value of the [**ProgressBar**](https://docs.microsoft.com/uwp/api/Windows.UI.Xaml.Controls.ProgressBar) control to 0.

If there are no selected items in the **ListView** control, then there is nothing to import. Otherwise, initialize a [**Progress**](https://docs.microsoft.com/dotnet/api/system.progress-1?redirectedfrom=MSDN) object to provide a progress callback which updates the value of the progress bar control. Register a handler for the [**ItemImported**](https://docs.microsoft.com/uwp/api/windows.media.import.photoimportfinditemsresult.itemimported) event of the [**PhotoImportFindItemsResult**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportFindItemsResult) returned by the find operation. This event will be raised whenever an item is imported and, in this example, outputs the name of each imported file to the debug console.

Call [**ImportItemsAsync**](https://docs.microsoft.com/uwp/api/windows.media.import.photoimportfinditemsresult.importitemsasync) to begin the import operation. Just as with the find operation, the [**AsTask**](https://docs.microsoft.com/dotnet/api/system?redirectedfrom=MSDN) extension method is used to convert the returned operation to a task that can be awaited, reports progress, and can be cancelled.

After the import operation is complete, the operation status can be obtained from the [**PhotoImportImportItemsResult**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportImportItemsResult) object returned by [**ImportItemsAsync**](https://docs.microsoft.com/uwp/api/windows.media.import.photoimportfinditemsresult.importitemsasync). This example outputs the status information to the debug console and then finallly, sets the cancellation token to null.

[!code-cs[ImportClick](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetImportClick)]

## Delete imported items
To delete the successfully imported items from the source from which they were imported, first initialize the cancellation token so that the delete operation can be cancelled and set the progress bar value to 0. Make sure that the [**PhotoImportImportItemsResult**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportImportItemsResult) returned from [**ImportItemsAsync**](https://docs.microsoft.com/uwp/api/windows.media.import.photoimportfinditemsresult.importitemsasync) is not null. If not, once again create a [**Progress**](https://docs.microsoft.com/dotnet/api/system.progress-1?redirectedfrom=MSDN) object to provide a progress callback for the delete operation. Call [**DeleteImportedItemsFromSourceAsync**](https://docs.microsoft.com/uwp/api/windows.media.import.photoimportimportitemsresult.deleteimporteditemsfromsourceasync) to start deleting the imported items. Us **AsTask** to convert the result to an awaitable task with progress and cancellation capabilities. After awaiting, the returned [**PhotoImportDeleteImportedItemsFromSourceResult**](https://docs.microsoft.com/uwp/api/Windows.Media.Import.PhotoImportDeleteImportedItemsFromSourceResult) object can be used to get and display status information about the delete operation.

[!code-cs[DeleteClick](./code/PhotoImport_Win10/cs/MainPage.xaml.cs#SnippetDeleteClick)]








 


