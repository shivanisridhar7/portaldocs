
<a name="working-with-data"></a>
# Working with data

<!-- required Overview document.  -->
    
<a name="working-with-data-overview"></a>
## Overview

The Azure Portal UX provides unique data access challenges. Many blades and parts may be displayed at the same time, each of which instantiates a new `ViewModel` instance. Each `ViewModel` often needs access to the same or related data. To optimize for these interesting data-access patterns, extensions follow specific patterns that consist of data management and code organization.

The following sections discuss these concepts and how to apply them to an extension.

* **Data management**

    * Shared data access using [data contexts](#data-contexts)

    * Using [data caches](#data-caches) to load and cache data

    * Memory management with [data views](#data-views)

* **Code organization**

    * Organizing the extension source code into [areas](#areas)

All of these data concepts are used to achieve the goals of loading and updating extension data, in addition to efficient  memory  management. For more information about how the pieces fit together and how the resulting construct relates to the conventional MVVM pattern, see the [Data Architecture](https://auxdocs.blob.core.windows.net/media/DataArchitecture.pptx) video that discusses extension blades and parts.

**NOTE**: In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory and  `<dirParent>`  is the `SamplesExtension\` directory. Links to the Dogfood environment are working copies of the samples that were made available with the SDK.

<a name="working-with-data-overview-data-contexts"></a>
### Data contexts

For each area in an extension, there is a singleton `DataContext` instance that supports access to shared data for the blades and parts implemented in that area. Access to shared data means data loading, caching, refreshing and updating. Whenever multiple blades and parts make use of common server data, the `DataContext` is an ideal place to locate data loading/updating logic for an extension [area](portalfx-extensions-glossary-data.md).

When a blade or part `ViewModel` is instantiated, its constructor is sent a reference to the `DataContext` singleton instance for the associated extension area.  In the blade or part `ViewModel` constructor, the `ViewModel` accesses the data required by that blade or part, as in the code located at `<dir>/Client/V1/MasterDetail/MasterDetailEdit/ViewModels/MasterViewModels.ts`.

```typescript

constructor(container: MsPortalFx.ViewModels.ContainerContract, initialState: any, dataContext: MasterDetailArea.DataContext) {
    super();

    this.title(ClientResources.masterDetailEditMasterBladeTitle);
    this.subtitle(ClientResources.masterDetailEditMasterBladeSubtitle);

    this._view = dataContext.websitesQuery.createView(container);
    
```

The benefits of centralizing data access in a singleton `DataContext` include the following.
* [Caching and Sharing](#caching-and-sharing)

* [Consistency](#consistency)

* [Fresh data](#fresh-data)

<a name="working-with-data-overview-data-contexts-caching-and-sharing"></a>
##### Caching and Sharing

  The `DataContext` singleton instance will exist for the entire amount of time that the extension is loaded in the browser. Consequently, when a blade is opened, and therefore a new blade `ViewModel` is instantiated, data required by the new blade will often already be loaded and cached in the `DataContext`, as required by some previously opened blade or rendered part.
  
  Not only will this cached data be available immediately,  which optimizes rendering performance and improves perceived responsiveness, but also no new **AJAX** calls are necessary to load the data for the newly-opened blade, which reduces server load and Cost of Goods Sold (COGS).

<a name="working-with-data-overview-data-contexts-consistency"></a>
##### Consistency

  It is very common for multiple blades and parts to render the same data in levels of different detail, or with different presentation. Moreover, there are situations where such blades or parts are displayed on the screen at the same time, or separated in time only by a single user navigation. 

  In such cases, the user expects to see all the blades and parts depicting the exact same state of the user's data. An effective way to achieve this consistency is to load only a single copy of the data, which is what `DataContext` is designed to do.

<a name="working-with-data-overview-data-contexts-fresh-data"></a>
##### Fresh data

  Users expect to see data that always reflects the current state of their data in the cloud, which is neither stale nor out-of-date. Another benefit of loading and caching data in a single location is that the cached data can be regularly updated to accurately reflect the state of server data. 

  For more information on refreshing data, see [portalfx-data-refreshingdata.md](portalfx-data-refreshingdata.md).

<a name="working-with-data-overview-data-contexts-developing-a-datacontext-for-an-area"></a>
#### Developing a DataContext for an area

Typically, the `DataContext` associated with a particular area is instantiated from the `initialize()` method of `<dir>\Client\Program.ts`, which is the entry point of the extension.

```typescript

this.viewModelFactories.V1$MasterDetail().setDataContextFactory<typeof MasterDetailV1>(
    "./V1/MasterDetail/MasterDetailArea",
    (contextModule) => new contextModule.DataContext());

```

There is a single `DataContext` class per area. That class is named `[AreaName]Area.ts`, as in the code located at  `<dir>\Client\V1\MasterDetail\MasterDetailArea.ts`.

```typescript

/**
* Context for data samples.
*/
export class DataContext {
   /**
    * This QueryCache will hold all the website data we get from the website controller.
    */
   public websitesQuery: QueryCache<WebsiteModel, WebsiteQueryParams>;

   /**
    * Provides a cache that will enable retrieving a single website.
    */
   public websiteEntities: EntityCache<WebsiteModel, number>;

   /**
    * Provides a cache for persisting edits against a website.
    */
   public editScopeCache: EditScopeCache<WebsiteModel, number>;

```

The `DataContext` class does not specify the use of any single FX base class or interface. In practice, the members of a DataContext class typically include [data caches](#data-caches) and [data views](#data-views).

<a name="working-with-data-overview-data-caches"></a>
### Data caches
 
  The `DataCache` classes are a full-featured and convenient way to load and cache data required by blade and part `ViewModels`. `DataCache` classes are specified in [portalfx-data-caching.md](portalfx-data-caching.md). 

* **CRUD methods**

  The CRUD methods are Create, Replace, Update, and Delete. Commands for  blades and parts often use these methods to modify server data. These commands should be implemented in methods on the `DataContext` class, where each method can issue **AJAX** calls and reflect server changes in associated DataCaches.

* **QueryCache**

  Loads data of type `Array<T>` according to an extension-specified `TQuery` type. `QueryCache` is useful for loading data for list-like views like Grid, List, Tree, or Chart.

The `DataCache` classes all share the same API. The usage pattern is as follows.

1.  In a `DataContext`, the extension creates and configures `DataCache` instances, as specified in [portalfx-data-caching.md](portalfx-data-caching.md). Configuration includes the following.  

    * How to load data when it is missing from the cache
    * How to implicitly refresh cached data, to keep it consistent with server state

     ```typescript

this.websiteEntities = new MsPortalFx.Data.EntityCache<SamplesExtension.DataModels.WebsiteModel, number>({
    entityTypeName: SamplesExtension.DataModels.WebsiteModelType,
    sourceUri: MsPortalFx.Data.uriFormatter(Util.appendSessionId(DataShared.websiteByIdUri), true),
    findCachedEntity: {
        queryCache: this.websitesQuery,
        entityMatchesId: (website, id) => {
            return website.id() === id;
        }
    }
});

``` 

1. In its constructor, each blade and part `ViewModel` creates a `DataView` with which to load and refresh data for the blade or part.

    ```typescript

this._websiteEntityView = dataContext.websiteEntities.createView(container);

```

1. When the blade or part ViewModel receives its parameters in the `onInputsSet` method, the ViewModel calls the  `dataView.fetch()` method to load data.

```typescript

/**
 * Invoked when the blade's inputs change
 */   
public onInputsSet(inputs: Def.BrowseMasterListViewModel.InputsContract): MsPortalFx.Base.Promise {
    return this._websitesQueryView.fetch({ runningStatus: this.runningStatus.value() });
}

```
  
For more information about employing these concepts, see [portalfx-data-masterdetailsbrowse.md](portalfx-data-masterdetailsbrowse.md).

<a name="working-with-data-overview-data-views"></a>
### Data views

Memory management is very important in the Azure Portal, because overuse by many different extensions has been found to impact the user-perceived responsiveness of the Azure Portal.

Each `DataCache` instance manages a set of [cache entries](portalfx-extensions-glossary-data.md), and the `DataCache` includes automatic mechanisms to manage the number of cache entries present at a given time. This is important because `DataCaches` in the `DataContext` of an `area` will exist in memory for as long as an extension is loaded, and consequently may support blades and parts that come and go as the user navigates in the Azure Portal.  

When a `ViewModel` calls the `fetch()` method for its `DataView`, this `fetch()` call implicitly forms a ref-count to a `DataCache` cache entry, thereby pinning the entry in the DataCache as long as the blade/part `ViewModel` has not been  disposed by the extension. When all blade or part `ViewModels` that hold ref-counts to the same cache entry are disposed indirectly by using via DataView,  the DataCache can elect to evict or discard the cache entry. In this way, the `DataCache` can manage its size automatically, without explicit extension code.
 
<a name="working-with-data-overview-areas"></a>
### Areas

From a code organization standpoint, an area can be defined as a project-level folder. It becomes quite important when data operations are segmented within the extension.

`Areas` are a scheme for organizing and partitioning of extension source code, making it easier to develop an extension with a sizable team. They impact how `DataContexts` are used in an extension. In the same way that extensions employ areas in a way that collects related blades and parts, each area also maintains the data that is required by these blades and parts. Every extension area gets a distinct `DataContext` singleton, and the `DataContext` typically loads/caches/updates the data of a few model types necessary to support the area's blades and parts.  

An area is defined in the extension by using the following steps.

  1. Create a folder in the `Client\` directory of the project that contains the extension. The name of that folder is the name of the area.
  1. In the root of that folder, create a `DataContext` named `<AreaName>Area.ts`, where `<AreaName>` is the name of the folder that was just created, as specified in [#developing-a-datacontext-for-an-area](#developing-a-datacontext-for-an-area). For example, the `DataContext` for the `Security` area in the sample is located at `\Client\Security\SecurityArea.ts`.  

A typical extension resembles the following image.

![alt-text](../media/portalfx-data-context/area.png "Extensions can host multiple areas")



<!--
    gitdown": "include-file", "file": "../templates/portalfx-data-caching.md"}

    gitdown": "include-file", "file": "../templates/portalfx-data-masterdetailsbrowse.md"}

    gitdown": "include-file", "file": "../templates/portalfx-data-lifetime.md"}

    gitdown": "include-file", "file": "../templates/portalfx-data-loadingdata.md"}

    gitdown": "include-file", "file": "../templates/portalfx-data-dataviews.md"}

    gitdown": "include-file", "file": "../templates/portalfx-data-projections.md"}
  
    gitdown": "include-file", "file": "../templates/portalfx-data-refreshingdata.md"}


    gitdown": "include-file", "file": "../templates/portalfx-data-virtualizedgriddata.md"}

    gitdown": "include-file", "file": "../templates/portalfx-data-typemetadata.md"}
-->

<a name="advanced-data-topics"></a>
# Advanced data topics

<!--    
    
<a name="advanced-data-topics-data-atomization"></a>
## Data atomization

Atomization fulfills two main goals: 

1. It enables several data views to be bound to one data entity, thus presenting a smooth and  consistent experience to the user.  In this instance, two views that represent the same asset are always in sync. 
1. It minimizes memory trace.

Atomization can be switched only for entities, which have globally unique IDs (per type) in  the  metadata system. In this case, a third attribute can be added to its `TypeMetadataModel` attribute in C#, as in the following example.

```cs

[TypeMetadataModel(typeof(Robot), "SamplesExtension.DataModels", true /* Safe to unify entity as Robot IDs are globally unique. */)]

```

The Atomization attribute is set to 'off' by default. The Atomization attribute is not inherited and has to be set to `true` for all types that should be atomized. 
In the simplest case, all entities within an extension will use the same atomization context, which defaults to one.
In this discussion, `<dir>` is the `SamplesExtension\Extension\` directory and  `<dirParent>`  is the `SamplesExtension\` directory. Links to the Dogfood environment are working copies of the samples that were made available with the SDK.

It is possible to select a different atomization context for a given cache object, as in the following example.

```cs

var cache = new MsPortalFx.Data.QueryCache<ModelType, QueryType>({
    ...
    atomizationOptions: {
        atomizationContextId: "string-id"
    }
    ...
});

```


 gitdown": "include-file", "file": "../templates/portalfx-<major-area>-bp-<topic>.md"  
gitdown": "include-file", "file": "../templates/portalfx-extensions-bp-data.md"}

<!-- optional FAQ document -->
<!-- gitdown": "include-file", "file": "../templates/portalfx-<major-area>-faq-<topic>.md"  -->
   ## Frequently asked questions

<a name="advanced-data-topics-data-atomization-"></a>
### 

* * * 
   
<!-- required Glossary document. -->
<!-- gitdown": "include-file", "file": "../templates/portalfx-extensions-glossary-<major-area>.md"  -->
   ## Glossary

 This section contains a glossary of terms and acronyms that are used in this document. For common computing terms, see [https://techterms.com/](https://techterms.com/). For common acronyms, see [https://www.acronymfinder.com](https://www.acronymfinder.com).

| Term                 | Meaning |
| ---                  | --- |
| area             |    |
| ARM | Azure Resource Manager | 
| COGS |   Cost of Goods Sold |
| CORS | | 
| CRUD |   Create, Replace, Update, Delete |
| JSON Web Token |  JSON-based token that asserts information between the server and the client.  For example, a JWT could assert that the client user has the claim "logged in as admin".  | 
| JWT token | JSON Web Token |
| lifetime | The amount of time an object exists in memory between instantiation and destruction.  It may be automatically destroyed by when a parent object is deallocated, or its existence may be managed by a lifetime manager object or a child lifetime manager object. |
| polling | The process that determines which client-side data should be automatically refreshed. |
| reactor | |