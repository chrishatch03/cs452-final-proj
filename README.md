# cs452-final-proj

**TA Note**
This codebase includes all resources necessary for submitting my final project report. The origional codebase I worked in contains api keys and code for hitting FamilySearch's api endpoints that belong to the BYU Family History Technology Lab and should be kept within the FHTL's private organization.


Project summary (1–3 sentences):
Kinarrative is an experience, an application that transforms your own raw FamilySearch ancestry data into interactive, narrated stories played back on an animated 3D globe right in your browser. It processes your thousands of ancestors through a carefully built data pipeline, discovering relatives, geocoding places, and generating structured narratives via OpenAI and then lets users explore their family history through map-driven playback with animations and frame-by-frame narration (natural langauge ai responses to the user's query).


Diagrams:
[New user login flow](new-user-login-flow.md)
[System Design Diagram](system-design.md)

Demo:
<video controls src="screen-recording-compressed.mp4" title="Demo"></video>

What did you learn?
1. This is the first time I've tried to build an app with a graceful UX that relies on slow and heavy backend operations. Kinarrative has to process, geocode, and organize 18,000 pieces of information for every new user, caching that info to quickly retrieve later is tough. And making the user's first experience with signing up for the app seamless is tricky. Deciding on the data pipeline for retrieving the user's ancestry was hard.
2. Using LLMs to produce structured data rather than just producing natural language text. I used OpenAI to generate strict JSON schemas for the payloads that were sent to the frontend web app. I also explored using generative UI to generate components for the frontend app as well.
3. I learned how tricky it is to generate svg graphics, especially in multi-frame animations. The app relies on displaying detailed genealogical data on maps and in stories, this was very hard and required multiple types of big frontend components.

Does your project integrate with AI? If yes, describe:
Definitely! Story and epic generation runs via OpenAI (GPT -4 Mini). After the backend services discover a user's ancestry, we use that model to build detailed JSON schemas to populate custom data structures and generate natural language text to answer the user's unique questions.

How did you use AI to build your project?:
I used Claude Code as a pair-programming partner throughout development, directing it to implement features, debug performance bottlenecks, and scaffold architecture while I made all major decisions, reviewed every change, and steered the overall direction of the product.

Why this project is interesting to you:
Kinarrative solves a problem close to home. My aunts and uncles have already gathered tons of names and records for thousands of my ancestors, so I have a hard time finding it enjoyable going back through my tree and looking for work to be done, it just seems like the successes to be discovered are sparce now (good thing, but makes family history work kind of boring for me). I don't even really know my family's story... do you? Kinarrative will empower users to be able to ask natural language questions about their genealogy using their own family's data from their FamilySearch account, and get lightning fast answers.

Explanation of failover strategy, scaling, performance, authentication, concurrency (if applicable):
The app caches computed overview data on database records to cut load time for returning users from 6 minutes to basically none, uses Dramatiq with Redis for background job processing so pipeline stages run concurrently while users interact with early results via SSE streaming, and includes failover strategies where geocoding cascades from FamilySearch to Nominatim and narrative generation falls back to templates if OpenAI is unavailable.  