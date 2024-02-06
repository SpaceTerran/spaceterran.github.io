---
title: "Slow Your Roll: A Thoughtful Approach to Adopting AI"
author: isaac
date: 2024-02-5 18:00:00 +0800
categories: [Artificial Intelligence (AI), OpenAI, ChatGPT, Ollama, Ollama WebUi, Microsoft, Sam Altman, Data Protection, Chatbot Development, Privacy Policies, AI Adoption, HR Considerations, Data Integration, Large Language Models (LLMs)]
tags: [AI, ChatGPT, OpenAI, Data Privacy, Sam Altman, Business Strategy, Chatbot Development, Microsoft Cognitive Services, Minimal Viable Product, Knowledge Base, AI Implementation, Organizational Efficiency, Future of AI, AI Ethics, AI Applications, GDPR, AI in Business, Ollama, Ollama WebUi]
render_with_liquid: false
toc: true
comments: true
---

So much hype has settled around OpenAI and its chatbot, ChatGPT, but the impact and implications of AI technology are still at the forefront for many organizations. I'll tell you now, AI is here to stay, and so is Sam Altman 🤣. Remember that crazy weekend in November 2023 when Microsoft seemed oblivious to Sam's firing and rehiring dramas? You'd think that after spending more than $100 million, they would have received a courtesy call. And yet, days later, he returned as co-founder and chief executive.

In case you missed it, Sam Altman, the CEO of OpenAI, was ousted from his position in November 2023 due to alleged mishandling of a significant breakthrough in their secretive Q* project. He was also criticized for his high-handed demeanor and profit-driven strategies that caused divisions amongst the board. He later returned. [(source)](<https://en.wikipedia.org/wiki/Removal_of_Sam_Altman_from_OpenAI>)

Now, onto the topic of this post - taking a moment before plunging into AI adoption. With countless vendors and consulting companies cropping up, trying to sell the latest AI services and tech, it's crucial to pause and focus on some key areas first.

Back in 2023, I was part of our AI innovation team at my current company. We found ourselves overwhelmed by the need to experiment with the latest technologies OpenAI had to offer (and others). Nonetheless, we assembled a small team of senior staff to focus on these critical areas:

1. Advanced analytics
2. Code generation
3. Communications
4. Knowledge base
5. Service desk augmentation
6. Content creation and ad creation
7. Security

My team focused on the knowledge base, service desk augmentation, and developing a secure, in-house ChatGPT application. As seasoned chatbot developers for our service desk, we believed in building our solution to handle data protection concerns more effectively than using existing, potentially privacy-infringing services. 

Our decision to build instead of buying primarily centered around **data protection!** If you're not aware already, data shared with most chatbots, even those seemingly harmless ones that adjust a picture's size or append a filter, earn revenues either upfront or at the back end by selling or reusing your data. 

Given OpenAI's initial absence of a clear privacy policy and Microsoft's emphasis on data privacy in their offerings, 

Per MS: 
>The user's prompts, completions, embeddings, and training data are kept private and not shared with other customers or OpenAI. This data is also not used to improve OpenAI models, Microsoft products or services, or automatically fine-tune Azure OpenAI models for the user's resource. Fine-tuned Azure OpenAI models are only available for the sole use of the customer who created them. [(source)](https://learn.microsoft.com/en-us/legal/cognitive-services/openai/data-privacy) 

We chose to opt-out from OpenAI and encouraged our team members to wait for our ChatGPT solution. We also addressed HR considerations, such as GDPR requirements and potential employment issues, ensuring that we could store and delete relevant data as necessary. 

Our objective was to launch a minimal viable product (MVP) within four months, complete with a custom web interface, file upload functionality, and the ability to search past conversations. We succeeded! A few team members worked tirelessly to integrate our custom frontend with Microsoft Cognitive Services using the Microsoft version of OpenAI's GPT-4(0314). Upon launch, it was immediately apparent that we had to extend our services to other Azure regions because of the demand. It was an exhilarating 72 hours after going live.

This experience taught me that although there were essential data integrations - like the ability to search our ticketing system or knowledge base to access our business-related information - the frontend investment wasn't entirely worth it. It'll serve your business better to find products that rapidly provide you with an interface and invest primarily in data integration. The models might not necessarily need to train on your data, but they should be capable of reading your data. While users might clamor for frontend improvements - I call these cost-of-living enhancements - such as an improved login experience, chat-saving, or internet browsing capabilities. However, these upgrades don't necessarily positively impact your organization.

OpenAI has since adjusted its policies. With their enterprise service, [https://openai.com/enterprise-privacy](https://openai.com/enterprise-privacy), they now state, 
>"You own your inputs and outputs (where allowed by law)". 

We'll see how this statement plays out 😉 but my point is they've developed an effective interface and are now focusing on allowing organizations to integrate with their business systems like SharePoint, custom APIs, and more. 

Ensure you don't compromise your data for speed, and always consider both short-term and long-term needs. Your long-term goal is to leverage AI to make decisions for your business, but what can AI do in the short term to improve employee efficiency? Examine tools like Microsoft Co-pilot and its built-in email writing and reading abilities. It's not about being inauthentic if you have AI help; it's about how much time you're saving by writing the core of the content in the tone you're looking for and using the AI to clean up grammar, spelling, etc. Or looking for duplicates in large data sets, or reading millions of log files to look for an issue with one of your servers, Summarize PDFs from government agencies like land and development, etc. These are things the AI can do today, with ease, and you don't need to build some complex tools.

There are even tools like Ollama and Ollama WebUI which can enable organizations to locally leverage open-source large language models (LLMs) without necessitating sizable infrastructure investments. 

Before you know it, you'll be discussing vector databases like Pinecone. However, first, focus on creating a minimal viable product and then build upon it.

Even in 2024, the landscape is still rapidly evolving. With incidents like the New York Times copyright litigation and safety issues involving deep fakes and voice-cloning scams, vigilance is required [(source)](https://data.berkeley.edu/news/what-experts-are-watching-2024-related-artificial-intelligence).

Of course, this guidance is subject to change. Slowing down to speed up means employing a mindful, strategic approach to integrate AI technologies into your business operations. By considering factors such as privacy, security, and long-term decision-making capabilities, organizations can effectively harness AI while avoiding potential pitfalls.


>Let's start a conversation below in the comments section about how we can effectively leverage AI while prioritizing data protection, security, and long-term decision-making capabilities. Whether you're an AI pro, a tech newbie, or just someone who loves a good digital drama – let's chat!
{: .prompt-tip }