# 6, Runtime View

## Overview

This shows the general flow of how the system will function.

```mermaid
sequenceDiagram

participant user as User
participant hub as Hub
participant db as HubDB
participant que as Service Bus
participant svc as Service
participant dm as Document Manager
participant fnt as Frontify

user ->> hub: 
rect rgb(240,240,240)
  note over user,fnt: Templates
hub ->> svc ++ #gold: Get Templates
svc ->> fnt ++ #gold: Get Templates
fnt -->> svc --
svc ->> fnt ++ #gold: Get Template Details
fnt -->> svc --
svc -->> hub --
end

hub ->> hub ++ #gold: Prepare Text and Images

hub ->> que  ++ #gold : Publish Brochure Artifacts 
hub -->> user --:done

    que ->> svc ++ #gold: Subscribe
loop
        svc ->> fnt ++ #gold: Load artifacts
        fnt -->> svc --
end
        svc ->> fnt ++ #gold : Generate Brochure
        fnt -->> svc -- : 
        svc ->> dm ++ #gold: Store
        dm -->> svc --: Saved
    svc ->> que --: Publish
hque ->> ub --++ #gold: Subscribe Brochure Metadata
hub ->> db : Add Brochure to attachments
hub -->> user --: Notify

note over user,fnt: 

user ->> hub ++ #gold
hub ->> dm ++ #gold: Access Brochure
dm -->> hub --: 
hub -->> user:

```

1. The selects if they want to generate a Bespoke or a Auto Generated brochure.
2. The service will make a Get Template request to Frontify, and filter the list to only those templates that are either the CSV or the Auto Generation brochure types.
3. The Get Templates method will return the template name and its detail.
4. Hub will create a service request object, and map the text and image artifacts to the template detail Keys.
5. Hub publishes the Brochure Generation request to the service bus.
6. The service picks up the brochure generation request, and creates a brochure object in the datastore.
7. If the User requested a Bespoke brochure then
   1. A CSV file is created.
   2. All of the image objects from the existing request brochure object are loaded into frontify and Assets are created.
   3. The Text and Asset Id's are written to the CSV.
   4. A request is made to the datastore to return all incomplete brochure objects.
   5. CSV rows are created for each.
   6. The CSV is uploaded.
   7. Once the Brochure has been created by the Frontify User, They will need to login to Hub and Upload the brochure, and indicate the generation is complete.
      1. Hub will inform the service that the brochure is complete.
      2. The service will complete the message in the datastore, stopping its inclusion in future CSV files.
8. If the User requested a Auto Generation Brochure then
   1. A Export Creative request is generated
   2. Either Poll Or Wait for a Webhook, which will indicate the generation is complete.
   3. Save the brochure in the Document Manager.
   4. Create a response message, and publish it onto the service bus.
