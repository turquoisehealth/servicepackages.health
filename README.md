
# Service Packages

  

Turquoise Health Standard Service Packages

  

## Overview

Turquoise Health’s repository detailing the methodology for creating Standard Service Packages viewable at https://www.servicepackages.health.

Standard Service Packages (SSPs) is an open-source project that seeks to simplify the communication of care across healthcare bills. Instead of communicating care across multiple bills, codes, and providers for a single encounter, SSPs wrap all charges and codes across bills into a single, easy-to-understand code.

  

We expect this project to support the Good-Faith Estimate portion of the No-Surprises Act and enhance the overall shoppability of healthcare. With this repository, we seek feedback on our methods for developing SSPs as well as thoughts on the initial release of data located in the “outputs” directory.

  

## Prospective Use Cases

SSPs are open-source, and open-ended. This lends them to a litany of use cases. Some of the applications that our team has been postulating include:

  

<h4> Support for Good-Faith Estimates related to the No-Surprises Act (NoSA)</h4>: NoSA requires providers to render cash-pay estimates to patients that are accurate to within $400 and include an itemization of all likely charges from all providers that may bill the patient. SSPs encapsulate both professional and facility charges, and itemize all procedures and services a patient may receive when undergoing the principal procedure.


<h4> Communication of care between providers and payers </h4>SSPs are 5-character alpha-numeric codes. This makes them compatible with 837 and 835 X12 files. There is potential for payers and providers to use SSPs as a new means of care communication and reimbursement that is as transparent, free and open source as the MS-DRG code set.

  

<h4> Allow patients to compare apples to apples </h4>Providers can bill different CPT codes alongside the same primary procedure which can make it difficult for a consumer to compare estimates taking different codes into account. SSPs remove that work from a consumer by resolving these into something more universal.

  
  

## Methodology

  

We began with **~2.7B claims from Komodo Health’s Sentinel product from 30M+ patients**.

<ul>

<li> ~2.2B professional claims</li>

<li>~450M institutional claims</li>

</ul>

  

We focused on isolating claims of interest which were defined as:

  

<ul>

<li> Institutional Claims </li>

<li>Claims billed with complex, high-cost procedures - These were captured by identifying claims with CPT/HCPCS codes that have J1 status indicators per CMS’s OPPS fee schedule

</li>

</ul>

  

The J1 status codes were then used as our “principal” procedures for each claim. We put claims into cohorts based on the presence of each principal procedure. For each claim in a principal procedure’s cohort, we inserted a record into a dataframe containing the code and charge level data for each claim line item (details on this can be found in the Claim Line Schema section of this README).

  

For each record created, we counted the frequency of every co-occurring code and calculated the associative percentage of each code with respect to the principal procedure. The calculation was performed as follows:

  

<ul>

<li> If principal procedure A occurred on 10 claims, and 7 of those claims contained co-occurring procedure B, the association of B to A is 7/10 or 0.7

</li>

<li>If principal procedure C occurred on 10 claims, and 5 of those claims contained co-occurring procedure B, the association of B to C is 5/10 or 0.5

</li>

</ul>

  

Every primary procedure has its own set of co-occurring codes and their corresponding associations. These calculations are the basis for our co-occurrence files found in the `output/` directory in this repository. The schema of the co-occurrence files are as follows:

  

| code | total_count | association | resolveFroms |
| ------ | ----------- | ------ | ---- |
| 0361 | 4 | 1.0 |  |
| 19083 | 3 | 1.0 |  |
| 0360 | 2 | 1.0 |  |
| 0401 | 2 | 1.0 |  |
| 88305 | 2 | 0.66 | 88306 |
| J3010 | 1 | 0.33 |  |

  

> We set the max association to be at 1.0 even if a code occurs multiple times on a claim

  
  
  

Many CPT/HCPCS codes are similar, report similar sorts of care, or are billed interchangeably. In order to strengthen associations between co-occurring codes and principal procedures, we leveraged NCCI PTP edits to resolve less frequently occurring codes into more frequently occurring codes. Counts and associations from less frequently occurring codes were added to the higher order codes with respect to NCCI PTP logic.

  

We repeated the exact process above for all professional claims to create a list of prospective co-occurring codes related to the institutional encounters with the principal procedures.

  

The team’s subject matter experts (SMEs) then manually reviewed a select listing of candidate SSPs that were deemed to be promising by an automated ranking process (detailed in the Candidate SSP Ranking section below). Once the SMEs deemed a candidate SSP to be production ready, the process for code inclusion on the https://www.servicepackages.health site was as follows:

* For institutional fees
   * Codes with >= .4 association are included in the Facility Fee section
	* Codes with association between .3 and .4 are included in the Optional Fee section
 * For professional fees
   * Codes with >= .5 association are included in the Professional Fee Section
  

After selecting production-ready SSPs, we needed to assign charge amounts to each co-occurring code to provide a general estimate of the cost of an SSP. Before calculating the per-code charges, we removed outlier claims from our data frames. We defined outliers as claims that had total charge amounts greater or less than two standard deviations from the median total claim charge amount.

  

The median total claim charge, in our filtered dataset of each claim containing a principal procedure, was stored in a dictionary. We then assign a charge weight to each co-occurring code as the average percentage of the claim charge when the co-occurring code appears alongside the principal procedure. The charge weight for each code in the SSP multiplied by the median total claim charge for the principal procedure to arrive at a code-level estimated charge.

  

### Claim Line Schema

  
  

| Field | Name | Type | Description |
| ----------- | ----------- | --------- | ----------- |
| claim_no | Claim Number | int | An identifier representing which claim a particular code belongs to |
| claim_line_no | Claim Line Number | int | Represents which line number on a claim |
| hcpcs_code | HCPCS Code | str | Five alphanumeric code representing procedure performed, HCPCS or CPT code |
| rev_code | Revenue Code | str | Four-digit code for a cost center within a hospital |
| charge | Charge | float | Billed amount for this line item |

  

A claim is composed of one or more of the above claim lines. When we processed the data, we created associations by looking at which `hcpcs_code`s and `rev_code`s occurred together on a particular `claim_line_no`.

  

In order to calculate associations, we transformed the standard claim schema above into a data frame where each row represented a single claim and all of its associated CPT/HCPCS codes, revenue codes, and line item charges.

  

An example schema, as well as a data dictionary for this dataframe, can be found below.

  

> In a future release, we will be incorporating Place of Service and Diagnosis codes into our associations (not represented in the claim line schema above).

  

### Claim Schema

  

| Field | Name | Type | Description |
| ----------- | ----------- | --------- | ----------- |
| claim_type | Claim Type | str | Claim type including institutional or professional claims |
| cpts | HCPCS/CPT codes | str | A comma-separated string of HCPCS/CPT codes found in the claim in order by claim line |
| revs | Revenue codes | str | A comma-separated string of revenue codes found in the claim in order by claim line |
| prices | Billed prices | str | A comma-separated string of billed prices for the corresponding HCPCS and rev codes |


A set of claims transformed into the SSP-ready dataframe would look like this:
> These are not real claims

| claim_type | cpts | revs | prices |
| ----------- | ----------- | --------- | ----------- |
| Institutional | 19083,G0416,88305 | 0401,0361,0361 | 3200,1000,500 |
| Institutional | 19083,88305 | 0360,0361 | 3000,500 |
| Institutional | 19083,G0416,J3010 | 0401,0360,0361 | 2000,1000,650 |

Our principal code here is 19083. All other codes and charges are considered to be co-occurring with respect to 19083.
 
Using a larger version of this data frame, our SSP code produces an output file per the process set forth in the methodology section above. The output file for this principal procedure can be found [here](https://github.com/turquoisehealth/servicepackages.health/blob/master/outputs/sorted_19083.csv).
  

## Candidate SSP Ranking

To isolate candidate SSPs, we computed aggregate statistics on each set of claims that contained a principal code after we removed claims that were deemed to be outliers. We plotted the distributions of each principal's set of claims with respect to their (1) claim length, and (2) CPT/HCPCS/revenue code charge variability.


Principal codes that had a claim population with relatively low claim length and charge amount variability ranked higher in our candidate SSP file.
  
  

## Reproduce

The Turquoise SSP creation scripts will be published in this repository at a later date. Our intent is for anyone to run this code on any claims database that is transformed to contain the required fields in the `claim_lines` and `claims` schemas above. It is important to keep in mind that output files from our code will vary from an independent run of this code as the claims data accessed by the code is likely to be distinct.

  

## Limitations

We are aware of limitations with this first release, and we’d be eager to hear your feedback on others. Here is a non-exhaustive list:

-   SSP development focused on claims data for hospital-based outpatient services. Thus, many of the current SSPs and output files skew towards hospital outpatient service archetypes. In further releases, we plan to involve a wider array of sites of care.
    
-   SSPs do not currently account for clinical differences between patients. Certain patients may require additional services with respect to a principal procedure and that may or may not be reflected in an SSP. In future iterations of SSPs, we plan to involve diagnosis as a qualifying criterion.
    
-   SSPs aren’t currently easily read by patients. This initial release is geared towards technicians. Through discussions on this repository, we hope to glean insights on the best way to create patient-friendly descriptions for SSPs and their associated codes/ charges.
    
-   SSPs are not episodic in nature, they are purely encounter based. As we develop the SSP library we will use individual SSPs as the building blocks to create generalized episodes of care.
    
-   There are many shoppable services that patients can receive during their course of care and we only felt that 50 SSPs were production ready for our website. We anticipate that interested parties will investigate the `outputs/` directory and provide guidance on additional SSPs that may be production ready.
    

  

## Outputs

In the `outputs/` directory, you can find the co-occurrence files that are produced from running the SSP code. We invite open exploration of these files and look forward to starting an industry-wide discussion of the validity of our approach and findings thus far.

> Important note: in the fourth column, you will see a list of CPT codes that were resolved from a particular CPT to the leftmost CPT using National Correct Coding Initiative Edits. For more details, visit https://www.cms.gov/Medicare/Coding/NCCI-Coding-Edits