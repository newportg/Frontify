# 5, Building Block View

```plantuml
@startuml System
!include <C4/C4_Component>
!include <azure/AzureCommon>
!include <azure/Compute/AzureFunction>
!include <azure/Web/AzureAPIManagement>
!include <azure/Integration/AzureServiceBusTopic>
!include <azure/Web/AzureSearch>

LAYOUT_WITH_LEGEND()
LAYOUT_LEFT_RIGHT()

AddElementTag("microService", $shape=EightSidedShape(), $fontColor="white", $legendText="micro service\neight sided")
AddElementTag("storage", $shape=RoundedBoxShape(), $fontColor="white")


System_Boundary(c1, "Knight Frank") {
    System_Boundary(auth, "Managed Identity") {
        System(hub, "Hub", "Internal Residential Transaction")
        System(ns, "NorthStar", "Hub Next")   

        AzureAPIManagement(apim, "APIM", "Management")
        AzureServiceBusTopic(sbus, "Serice Bus Topic", "Integration")
        Container_Boundary(app, "Address Service", "Allows users to read address information", $tags = "microService") {
            AzureFunction(aFunc, "Function", "Http/Sbus Trigger")
            Component(compDom, "Domain Logic Component", "Process, Rules and validation")
            Component(compSec, "Security Component", "Provides functionality to validate callers and grant access to the service ")
            Component(compCosmos, "CosmosDB  Component", "Provides db CRUD, versioning functionality")            
            Component(compFrontify, "Frontify Service Handler", "Provides functionality to interact with the Frontify service")
            Component(compAudit, "Audit Component", "Provides functionality that is useful for administration")
        }
        SystemDb(cosmos, "CosmosDB", "Content Managemnt Storage", $tags = "storage")
        AzureSearch(search, "AzureSearch", "Provides search functionality")
    }
}

System_Ext(extFrontify, "Frontify", "External Brochure Generation Service")

Rel(hub, sbus, "Searches and selects address", "Http")
Rel(ns, sbus, "Searches and selects address", "Http")

Rel(apim, aFunc, "")
Rel(sbus, aFunc, "")

Rel(aFunc, compDom, "")
Rel(aFunc, compSec, "")
Rel(compDom, compCosmos, "")
Rel(compDom, compFrontify, "")
Rel(compDom, compAudit, "")

Rel(compCosmos, cosmos, "Get previously cached brochure Detail")
Rel(compCosmos, search, "Search brochure detail")
Rel(cosmos, search, "")
Rel(compFrontify, extFrontify, "Hierarchical search finding addess", "Https")

@enduml
```

This service has been designed to handle the interaction with the Frontify brochure generation service. Based on the information supplied the service will determine whether to use the, more automated API service, or the more bespoke CSV file service, which requires a great deal of user interaction.

The service will provide auditing information, which will return information based on the quantity of requests by day/month, and by whom.

### Contained Building Blocks

| **Name**                 | **Responsibility**                                                                       |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| Azure Function           |  Is the interface to all systems. provides both Restful API and Service Bus Interfaces.  |
| Security Component       |  Provides all Managed Identity and User Identity/Access controls                         |
| Audit Component          | Provides a standardised set of audit data retrieval methods, and SOC Alert rules         |
| Cosmos Db Component      | Provides any abstractions or specialisations necessary to access Knight Franks Cosmos DB |
| Frontify Service Handler | Is a specific interface for the Frontify Brochure Generation service.                    |
| Domain Logic Component   | Provides rules and workflow to satisfy the requirements of this service                  |

### Azure Function

The Azure function is the external interface for the service and will support both a RESTful API and a service bus interfaces.

The Service Bus interface is the main interface for creating brochures.

The RESTful API will follow standard REST naming conventions e.g. /Customers or /Customer/{id} to return a collection or a single entity. The RESTful API also employ OPENAPI documentation frames detailing the request and response models, and all return statuses.

## Components

The component libraries should be general and built so they can be copied or included in other projects.

### Security Component

This component will validate the Managed Identity Header. This indicates that the calling application has been authorised to use this service. If a caller fails the Managed Identity authorisation then the method should return a HTTP Status of 401 Unauthorised. If a user token is included, then it should be validated with Entra Id(Active Directory) and the returned claims inspected. Failure should result in a HTTP Status 403 Forbidden.

### Audit Component

Generally all audit components will implement the same API calls, this will allow an admin application to produced consistent reporting across the service estate. The Audit component can supply additional API methods to return specific information that only applies to the current service. The audit interface should be included with the service API, and include calls to return counts based on usage by user/system.

### Cosmos Db Component

This component should utilise the CQRS pattern, All read requests should be serviced by Azure Search queries. Cosmos Db should be implemented with the NoSQL container. This component should use both the Microsoft.Azure.Cosmos and Azure.Identity libraries. The calling client application will be authorised by Managed Identity to access the Cosmos DB. This component should encapsulate any Knight Frank nuances which are general to our usage of Cosmos DB, such as naming conventions or setting configuration.

### Frontfy Service Handler

This service handler will interact with the Frontify Brochure Generation system. All data will be passed to/from this service in a canonical form, so internal systems are not dependant on changes to the Frontify service.

### Domain Logic Component

The domain logic component implements the business requirements of the service. It consists of a workflow and rules. Workflow will not contain rules. The workflow will only ever ask questions of the rules.

### Sequence of Interactions

The sequence diagram below shows the flow calls to Frontify.

The group in red is an area of discussion, as this is the manual, bespoke, process, so the service will not be able to track changes or completion. Meaning the incomplet objects in the datastore will keep building, making the CSV larger and larger.

```plantuml
@startuml

actor "Hub User" as user
participant Hub as hub
database HUbDb as hubdb
queue "Service Bus" as bus
participant Service as svc
database CosmosDb as db
participant "Document Manager" as dm
participant Frontify as fmt
actor "Frontify User" as fmtUsr

user -> hub ++ #gold
  hub -> svc ++ #gold: Get Templates
    svc -> fmt ++ #gold: Get Templates
    fmt --> svc --
  svc --> hub --
hub --> user --: Select template

user -> hub ++ #gold: Create Brochure
  hub -> bus ++ #gold: Publish Create Brochure
    bus -> svc ++ #gold: Subscribe Brochure Artifacts
    Group Create Brochure Object
        svc -> db : Create Brochure Object
        svc -> db : Add Text to Brochure Object
        loop
            svc -> fmt ++ #gold: Load and Create Asset
            fmt --> svc --  
            svc -> db : Add AssetId to Brochure Object
        end
    End
    Group #pink Bespoke 
        svc -> svc : Create CSV
        svc -> db : Get Incomplete Brochure Objects
        svc -> svc : Create CSV rows
        svc -> fmt ++ #gold: Upload CSV
        
        fmt -> fmtUsr ++ #gold 
        fmtUsr --> fmt --
        fmt --> svc --
        ...
        Group Brochure Completion / User
          fmtUsr -> hub : Set Brochure Complete
          hub -> svc : Set Brochure Complete
          svc --> hub
        End

    End
    Group #lightblue Auto
        svc -> fmt ++ #gold: Create Brochure Request from Brochure Object
        fmt -> fmt : 
        ...
        Group Brochure Completion /  WebHook
            fmt -> svc --: Brochure Created
        End     
    End

    svc -> fmt ++ #gold: Get Brochure
    fmt --> svc --

    svc -> dm ++ #gold: Save Brochure
    dm --> svc --
    svc -> db : Complete Brochure Object
    svc --> bus --: Publish Brochure metadata

  bus -> hub --: Subscribe Brochure metadata
  hub -> hubdb ++ #gold: Save metadata
  hubdb --> hub --: 
hub -> user --: Notify
@enduml
```

### Flowchart

```plantuml

start
fork
  : 1, Get Templates;
  Group Hub
    :HUB : Get Templates;
  EndGroup
  Group Service
    :SVC : Get Filterd Templates and details from Frontify;
  EndGroup

fork again
  : 2, Generate Brochure;

  Group Hub
    :Hub : Generate Text and Images;
    :1, Get Template;     
    
    if ( Bespoke? ) then (TRUE)
      :Service Request : Bespoke Flag Set FALSE;
    else (FALSE)
      :Service Request : Bespoke Flag Set TRUE;
    endif
    :Service Request : Set TemplateId;
    :Service Request : Map Text and Images\n into Svc Request with Template Keys;
  Endgroup

  Group Service Bus
    :Pass Service Request;
  Endgroup
  Group Service
    Group Create CosmosDB Brochure Object
    :SVC : Create a Brochure Object in Datastore;
    :SVC : Store Text in Brochure Object;

      Group Asset Upload
        repeat
            :Upload Image to Frontify;
            :Save AssetId in Brochure Object;
        repeat while (More images to upload?) is (yes) not (no)
      EndGroup
    EndGroup

    if (Generate Bespoke Brochure?) then (TRUE)
      Group Generate CSV
        :Get All incomplete BESPOKE Brochures from Datastore;
        :Create CSV;
        repeat
          :Get Brochure Object;
          :CSV: New Row;
          :CSV: Add Texts to matching column heading;
          :CSV: Add AssetIds to matching column heading;
        repeat while (More Brochure Objects?) is (yes) not (no)

        :Upload CSV;
        :User : Generate Brochure;
        :3, Complete Brochure; 
      EndGroup

    else (FALSE)
      Group Auto Generate
        :Using Current Brochure Object;
        :ExportCreative and Associate Assets and Template;
        :Generate Brochure - Returns Brochure Id;
      EndGroup
    Group CleanUp
      :Check Brochure Status;
      :Get Brochure;
      :Store Brochure;
      :Complete Datastore Brochure Object;
    EndGroup

    Group Service Bus
      :Return Brochure Metadata;
    Endgroup
    endif
  EndGroup

fork again
  : 3, Complete Brochure;
  Group Hub
    :HUB : Complete Brochure;
  EndGroup
  Group Service
    :SVC : Mark datastore Brochure Object Complete;
  EndGroup
endfork
stop

```

### Component Detail

1. Azure Function Detail
2. Security Component
3. Service Component
4. Domain Logic Component
5. Cosmos DB Component
