---
layout: post
title:  "Finding cluster bombs in your retirement savings"
author: Caleb Tutty
date:   2017-03-23 08:20:42 +1300
categories: data-journalism
tags: kiwisaver data news weapons tobacco
---

A technical walk-through behind the [Herald investigation](http://insights.nzherald.co.nz/article/kiwisaver-investments) into Kiwisaver investments in cluster munitions, landmines and nuclear weapons.

In August 2016 the New Zealand Herald and Radio New Zealand broke stories around Kiwisaver investments in assets blacklisted by the Superannuation fund. I worked with Matt Nippert, analysing every asset declared in the Financial Market Authority's consolidated Kiwisaver annual disclosure files and presented the data in an [interactive visualisation](http://insights.nzherald.co.nz/article/kiwisaver-investments).

### Prelude to exclude

Matt Nippert investigated the Superfund in a piece in [the Listener](http://www.noted.co.nz/archive/listener-nz-2007/dirty-dollars/) back in 2007. The Superfund then created a [list of investments](https://www.nzsuperfund.co.nz/how-we-invest-responsible-investment/exclusions) which the fund governors have decided to exclude.

The Superfund's exclusion list is available as an Excel spreadsheet, which was our starting point. The current (September 2016) version is available as a [download](https://www.nzsuperfund.co.nz/sites/default/files/documents-sys/NZSF%20RI%20Exclusions%20List%20Sept%202016.xlsx) from the [exclusions page](https://www.nzsuperfund.co.nz/how-we-invest-responsible-investment/exclusions).

At the time of analysis the list was as follows:

|Superfund Exclusion Category     										 |	Companies																									  |
|------------------------------------------------------|--------------------------------------------------------------|
|Anti-personnel mines       											     | Northrop Grumman Corporation                                |
|Cluster munitions      													     | Alliant Techsystems Inc, General Dynamics Corp, Textron Inc |
|Nuclear Explosive Devices and Nuclear Base Operators  | AECOM, BWX Technologies, Fluor Corp, Honeywell International, Huntington Ingalls Industries Inc, Jacobs Engineering Group Inc, Lockheed Martin Corp, Serco Group PLC |
|	Tobacco																							 | Alliance One International Inc, Altria Group Inc, British American Tobacco, Eastern Tobacco, Grupo Carso SAB de CV, Grupo Sanborns, S.A.B. de C.V., Huabao International Holdings, Imperial Tobacco, Japan Tobacco Inc, Kirin Holdings Co Ltd, KT&G Corp, Lorillard Inc, Philip Morris, Reynolds American, Schweitzer-Mauduit International, Souza Cruz SA, Swedish Match AB, Torii Pharmaceutical Co Ltd, Universal Corp/VA, Vector Group Ltd |
|	Other																								 | Barrick Gold Corp, Elbit Systems Ltd, Freeport McMoRan Inc, KBR Inc, Shikun & Binui Ltd, Tokyo Electric Power Company, Incorporated, Zijin Mining Group Co Ltd, Acacia Mining |


### Gathering documents

To analyse which KiwiSaver schemes included investments in these companies requires a splash of programming. The aim is to match the assets present in FMA disclosures for each fund with these blacklisted Superfund assets.

The Financial Markets Authority receive and collate [quarterly and annual reports](https://fma.govt.nz/consumers/kiwisaver/more-comprehensive-kiwisaver-details/) from KiwiSaver scheme providers in a data file format [prescribed in legislation](http://www.legislation.govt.nz/regulation/public/2013/0047/latest/DLM5094045.html).

**This process has now changed, with a new register to be hosted by the Companies Office, called [Disclose](https://www.companiesoffice.govt.nz/disclose).**

Visit the the FMA ['More comprehensive KiwiSaver details' page](https://fma.govt.nz/consumers/kiwisaver/more-comprehensive-kiwisaver-details/) and [download](https://fma.govt.nz/assets/Spreadsheets/KiwiSaver/FMA-KDS-WEB-Annual-20160331.xlsb) the most recent Annual Disclosure Statement.

The rest of this blog post will talk about how to get the data out of this spreadsheet (named `FMA-KDS-WEB-Annual-20160331.xlsb` at time of writing) and into an aggregation grouped by Superfund exclusion category.

#### When it comes to analysis, only some Excel

This spreadsheet may seem incomprehensible at first.

Getting a handle on terms is a useful first step: individuals choose a membership KiwiSaver `fund`, which is part of a `scheme`. An example is that an individual may choose the `Balanced fund` offered as part of the `Westpac KiwiSaver scheme`. One word of warning: individuals may opt to split across a number of funds, so we may have to refer to 'memberships' rather than counts of people.

The most relevant sheets are `submitted_disclosures` and `Assets`. They both have a corresponded table which describes the column headers in human readable terms: `Submitted_disclosures_desc` and `Assets_desc` (`_desc` being description).

The `submitted_disclosures` sheet holds the id field to map assets back to particular funds.


##### Submitted Disclosures

!['submitted_disclosures' tab](http://i.imgur.com/7ChRPLE.png)


KiwiSaver funds disclosures each have a unique identifier, or [UUID](https://www.wikiwand.com/en/Universally_unique_identifier) in the column headed `KSDID`. This looks something like `15DEA429-2A3C-E611-845D-005056BC4DA1` (which is ANZ KiwiSaver Scheme's Conservative Fund). We want to match these IDs with each of the 97,462 individual assets declared by each fund.

##### Assets

!['Assets' tab](http://i.imgur.com/GSNAv6G.png)

The `Assets` tab lists every single investment. We assume ASB would probably avoid investing in `1MDB Global Investments Ltd` in future.


#### Avoiding pitfalls

From experience the quarterly reports only include the top 10 investments, and as such aren't useful. Annual Disclosure Statements provide all assets invested by each fund - as well as other interesting performance measures.

My first attempt looked at the [ISIN](https://www.wikiwand.com/en/International_Securities_Identification_Number) column present in the Superfund exclusion spreadsheet. I thought that this identifier would provide an easy matcher with the `s_code` column in the `Assets` sheet, making the process nice and easy. Unfortunately this data is inconsistent, with large gaps and malformed IDs - rendering this technique useless.

The FMA also warns:

> As the FMA consolidates the data files submitted by KiwiSaver managers, it's important to note we do not make any representation or give any assurance of the accuracy, quality or completeness of the consolidated data. KiwiSaver managers are each responsible for the accuracy, quality and completeness of their individual data.

### Putting it all together

In doing this analysis, I looked at all three years available (2014, 2015, 2016) but we only visualised and reported on 2016 data. Using a script like the following, I joined the assets with fund names and was able to calculate the amounts in each exclusion category.

To work out the figures for each investment (with the caveat of precision errors arising from the number of reported decimal places), I used the `Percentage of Fund Net Assets` with the total fund value to calculate the value of that asset.

```ruby
require 'csv'

[2014, 2015, 2016].each do |year|

  funds = CSV.read "processing/#{year}.csv", headers: true
  disclosures = CSV.read "processing/#{year}-disclosures.csv", headers: true

  new_csv = CSV.open("processing/#{year}-joined.csv", 'wb')

  new_csv << ['id', 'fund_name', 'scheme_name', 'percentage', 'total_value',
              'investment_value', 'asset_name', 'exclusion_reason']

  funds.each do |row|
    next unless row['exclusion_reason']

    scheme = disclosures.find {|a| a['KSDSID'] == row['KSDSID'] }
    new_csv << [row['KSDSID'], scheme['s_KSFundName'], scheme['s_KSSchemeName'], row['p_ofFundNetAssets'].to_f,
                scheme['n_totFundValue'].to_f, (scheme['n_totFundValue'].to_f * (row['p_ofFundNetAssets'].to_f / 100)),
                row['s_AddAssetName'], row['exclusion_reason']]

  end

end
```

I moved from Excel to CSV files, but this data set required a large amount of janitorial data tasks. Asset names required careful scrutiny to ensure that all valid permutations were included, and deceptively similar (but unrelated) names were not.


### Reading The Blacklist

Cluster munitions, anti-personnel mines and nuclear weapons feature on the list of assets excluded from the Superfund, and can also be found in legislation, such as the following:

The [Cluster Munitions Prohibition Act 2009](http://www.legislation.govt.nz/act/public/2009/0068/latest/DLM2171670.html), enacted after the signing of the Cluster Munitions Convention, states:

> A person commits an offence who provides or invests funds with the intention that the funds be used, or knowing that they are to be used, in the development or production of cluster munitions.

The [Anti-Personnel Mines Prohibition Act 1998](http://www.legislation.govt.nz/act/public/1998/0111/latest/DLM17844.html) describes assisting or inducing as an offence:

> No person may—
> (a) use an anti-personnel mine; or
> (b) develop, produce, or otherwise acquire an anti-personnel mine; or
> (c) possess, retain, or stockpile an anti-personnel mine; or
> (d) transfer to anyone, directly or indirectly, an antipersonnel mine; or
> (e) assist, encourage, or induce, in any way, anyone to engage in conduct referred to in paragraphs (a) to (d).

[Nuclear weapons legislation](http://www.legislation.govt.nz/act/public/1987/0086/latest/DLM115141.html#DLM115141) includes language around prohibiting New Zealand citizens aiding and abetting manufacture of 'any nuclear explosive device'.


Tobacco manufacturing, the processing of whale meat, and unethical practices (some potentially described as human rights violations) can also blacklist an asset from Superfund investment.


Hopefully this blog post is helpful to other journalists in the future. Please feel free to [contact me](https://twitter.com/Caleb_T) with any questions.
