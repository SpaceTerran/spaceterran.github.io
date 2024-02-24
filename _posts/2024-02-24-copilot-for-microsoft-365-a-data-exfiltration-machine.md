---
title: "Copilot for Microsoft 365, A Data Exfiltration Machine?"
author: isaac
date: 2024-02-24 07:00:00 -0700
categories: [Artificial Intelligence (AI), Data Protection, Privacy Policies, HR Considerations, Data Integration, Business Strategy, AI Adoption, AI Implementation, Security]
tags: [AI, Data Privacy, Business Strategy, Chatbot Development, Microsoft Cognitive Services, AI Implementation, Knowledge Base, Organizational Efficiency, Future of AI, GDPR, AI Ethics, AI in Business, Advanced Analytics, Communications, Service Desk Augmentation, Content Creation, Security]
render_with_liquid: false
toc: true
comments: true
image:
  path: /assets/img/myphotos/copilot-for-microsoft-365-data-exfiltration-machine.jpg
  alt: Copilot for Microsoft 365 Data exfiltration Machine?
---

So, you have that shiny new MS Copilot license costing $30 per user, per month, and presumably, your users couldn't be happier. The product certainly comes with an impressive array of features. I particularly enjoy the integration with MS Teams, where it's possible to query past conversations from various chats and emails. For instance, imagine you're trying to recall the key dates for one of your projects. MS Copilot will search multiple sources and, in most cases, provide a relevant suggestion or the exact information you're seeking. It will even reference where it found the data, such as in an email or a file attachment. There are numerous examples similar to this. However, I'd like to use a few moments of your time to discuss the "web content" aspect and the numerous plugins that are available for use with MS Copilot.

Let's start by addressing the web content feature. This tool allows MS Copilot to search web content while answering prompts. My immediate reaction was, "How are these queries handled? Does it remain within my tenant? Is my company data staying within our tenant?" To find out, let's examine this link: [Microsoft Copilot for Microsoft 365 and web content](https://learn.microsoft.com/en-us/microsoft-365-copilot/microsoft-365-copilot-privacy#microsoft-copilot-for-microsoft-365-and-web-content).

The primary concern lies around the potential inclusion of confidential company data within web search queries made through the Bing Search API. Detailed in these lines:

>"Only the search query, which is abstracted from the user's prompt and grounding data, goes to the Bing Search API outside the boundary."
{: .prompt-info }

and

>"However, they may still include some confidential data, depending on what the user included in the prompt."
{: .prompt-info }

Also, this statement:

>"The Microsoft Products and Services Data Protection Addendum (DPA) doesn’t apply to the use of Bing." 
{: .prompt-info }

This has the potential to be worrying, especially when considering that the data protection standards utilized by Bing may differ from those in place in the Microsoft 365 environment.

From my perspective, this is concerning. If we value our company data and consider even the prompts we ask as potential assets that may provide competitive advantages in the future, there's a risk present here. The situation reminds me of the news on [Samsung data leaked on ChatGPT](https://www.businesstoday.in/technology/news/story/samsung-employees-accidentally-leaked-company-secrets-via-chatgpt-heres-what-happened-376375-2023-04-06#:~:text=the%20AI%20chatbot.-,Samsung%20data%20leaked%20on%20ChatGPT,notes%20relating%20to%20their%20hardware.)

>Samsung had allowed its engineers at the semiconductor division to use ChatGPT to help fix problems with source code. The employees mistakenly entered top-secret data like source code for a new program, and internal meeting notes relating to their hardware.
{: .prompt-info }

Looking ahead, I suspect there will be numerous modifications to these guidelines. There may even be potential options for allowing certain users to use this feature -- a stark contrast to the current binary state where it's either entirely on or off. However, these points are worth considering when integrating MS Copilot into your organization. If, for now, you need to disable this feature, follow the instructions here: [Choose whether Microsoft Copilot for Microsoft 365 can access web content](https://learn.microsoft.com/en-us/microsoft-365-copilot/manage-public-web-access#choose-whether-microsoft-copilot-for-microsoft-365-can-access-web-content).

As I'm wrapping up, let's discuss the array of plugins readily available in MS Teams that are "MS Copilot Ready." If data security is paramount to you, you may have already set these plugins to be administratively managed. However, it's easy to miss this important step. I would recommend making these plugins admin managed, which allows for your Data Governance teams to ascertain how and what each plugin does with your data upon utilization. If MS Copilot reaches out to Jira to retrieve Jira Stories, for instance, and you're a Jira user, this may not concern you because you already have contracts and agreements in place to guarantee the ownership of your data. But what about plugins like this [Meeting Summaries from Read AI](https://appsource.microsoft.com/en-us/product/office/WA200003896?tab=Overview) one? 

Let me clarify here— I'm not labeling this company as unfavorable or even suggesting to avoid it. However, if one of your employees stumbles across this plugin, incorporates it into their work process, what guarantees do you have that your data isn't being used either to train their models or, worse yet, for them to be reselling your data? Now, Read AI may not have these issues, and I don't have any experience with the tool either, but it's merely an example of a MS Copilot plugin readily available that may not be desirable.

The recommendation here, as I stated in my last post about AI... "Slow Your Roll." While it's tempting to try and utilize all these plugins and integrations, it's easy for a "plugin" to potentially exfiltrate data.

What challenges are you running into? Are you having these discussions about data security with these new tools? Let's chat about it in the comments below!