---
title: Integrating with Azure Search
tags: [azure, search, integration]
image:
  feature: header/h06.svg
---
I currently help maintain a site called [Azure Citadel](https://azurecitadel.com), which is host to lots of techie articles, guides, labs and things related to Azure

Azure Citadel is in part is a big reason why this blog has been dead for 12 months, as the majority of my technical writing ended up there rather than as blog posts. Sorry!

<!--more-->

{% include img src="citadel.png" link="https://azurecitadel.com" %}

We recently did a complete overhaul of the Azure Citadel site and part of this was to improve the search solution we had. In the past we simply linked to Google site search (e.g. the search for `site: azurecitadel.com foo`) which was a really poor way of doing things, and totally reliant on Google indexing the site

It made sense to try something better, and it also made sense to use Azure Search, after all the site was all about Azure and Azure services! [Azure Search](https://azure.microsoft.com/en-gb/services/search/) is one of the many platform services available in Azure, in this case providing a turn-key "search-as-a-service" solution. I knew getting Azure Search to work with Citadel wouldn't be particularly trivial. Firstly the site is completely static (using Jeykll) so there's no simple database of posts or content to index. Also the other challenge would be creating the frontend search pages on the site, I'd never done this before, so had no idea how much work would be involved.

But it quietly bugged me for a while, and I wanted to learn more about Azure Search for a hack I had coming up so I decided to give the integration a go. After some hair pulling, a fair bit of swearing and a couple of rethinks on the approach - I think I've cracked it!

I've written up all the details of the integration over on Azure Citadel (yes that's a bit meta/Inception but hey!), hopefully in such a way the solution can be reused by others

[Read Full Article &nbsp; <i class="far fa-book"></i>](https://azurecitadel.com/data-ai/azure-search-integration/){: .btn .btn-success .btn-lg .centered}