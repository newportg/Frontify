# 7, Deployment View

```plantuml
@startuml System
!include <C4/C4_Component>
!include <azure/AzureCommon>
!include <azure/Compute/AzureFunction>
!include <azure/Web/AzureAPIManagement>
!include <azure/Integration/AzureServiceBusTopic>
!include <azure/Networking/AzureApplicationGateway>
!include <azure/Networking/All>
!include <azure/Web/AzureWebApp>
!include <azure/DevOps/AzureApplicationInsights>
!include <azure/Storage/AzureBlobStorage>
!include <azure/Databases/AzureCosmosDb>
!include <azure/Security/AzureKeyVault>
!include <azure/Identity/AzureAppRegistration>
!include <azure/Identity/AzureActiveDirectory>
!include <azure/Identity/AzureManagedIdentity>

LAYOUT_WITH_LEGEND()
'LAYOUT_LEFT_RIGHT()

AddElementTag("microService", $shape=EightSidedShape(), $fontColor="white", $legendText="micro service\neight sided")
AddElementTag("storage", $shape=RoundedBoxShape(), $fontColor="white")

System_Boundary(s1, "Knight Frank") {

    AzureWebApp(local, "Application", "Internal")
    AzureActiveDirectory(ad, "", "Active Directory")
    AzureManagedIdentity(miapp, "Managed Identity", "route")

    Container_Boundary(c1, "Shared Subscription") {

        Container_Boundary(rgsd, "rg-shared-data-env-ne") {
            AzureCosmosDb(db, "cosmos-shared-data-env-ne", "Container - Frontify")
        }

        Container_Boundary(rgsh, "rg-shared-env-ne") {
            AzureAPIManagement(apimI, "apim-shared-env-ne", "Internal Interface")
            apimI -up-> ad : Validate App Registration
        }

        AzureManagedIdentity(misvc, "Managed Identity", "route")
        AzureAppRegistration(reg, "", "Application Registration")

        Container_Boundary(rggw, "rg-frontify-env-ne") {
            AzureFunction(svc, "func-frontify-env-ne", "Azure Function")
            AzureApplicationInsights(ai, "appi-frontify", "Application Insights")
            AzureBlobStorage(st, "stFrontifyEnvNe", "Atorage Account")
            AzureKeyVault(key, "key-frontify-env-ne", "Key Vault")
            

            svc --> ai : uses
            svc --> st : uses
            svc --> key : uses
            svc -up-> reg : register
        }

        svc --> db : uses
        apimI -> svc 
        local ---> apimI

        apimI -down-> misvc : register
        svc -up-> misvc : validate
    }

    rgsh -[hidden]- rggw
    rgsh -[hidden]- rgsd

    local -down-> miapp : register
    apimI -up-> miapp : validate
    ad -down-> reg : validate

}

System_Ext(extFrontify, "Frontify", "External Brochure Generation service")
svc --> extFrontify

@enduml
```

## Security

The application is accessed by Internal web applications.

* Internal are within the Knight Frank network.
* Not accessible from the public internet.
* Managed Identity will be used between APIM and the Azure Function Service.
* The Azure Function will validate the MI token.

## Application

* The Application resource group will contain the usual components for a microservice.
* The components will follow our naming convention.

| Component            | Name                 |                                                |
| -------------------- | -------------------- | ---------------------------------------------- |
| Azure Function       | func-frontify-env-ne |                                                |
| Application Insights | appi-frontify-env-ne |                                                |
| Storage Account      | stfrontifyEnvNe      | Camelcase for readability, should be lowercase |
| Key Vault            | key-frontify-env-ne  |                                                |

**N.B. Env should be changed to match the deployment environment.**

The Application will be registered in Active Directory (Entra).

## DataStore.

* The application uses a Cosmos Database as its data store.
* A new cosmos DB container of NOSQL type should be set up.