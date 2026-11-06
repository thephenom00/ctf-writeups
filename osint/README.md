# OSINT - Trip
## Challenge Info
| Field | Value |
| --- | --- |
| **Context** | CTF class at CTU in Prague, 2026 |
| **Challenges** | Trip |
| **Category** | OSINT - Image Geolocation and Route Reconstruction |
| **Objective** | Identify a hiking signpost in Šumava and retrieve the identification code of its top sign |
| **Tools Used** | Google reverse image search, mapy.com, LLM, hiking websites and public photographs |

## Task Setup

We went on a trip to Šumava and took a few photos. It was nice, we walked up, enjoyed the view, regained some energy by having a slightly late lunch and started our way back, which was perhaps even more beautiful.

Your task is to find out which hiking signpost we passed on the way back at around 3 PM. Submit the identification code of the top sign on the signpost as the flag.

## Task Solution

The first step was to identify the location. A big hint was that it was somewhere in Šumava. Using Google reverse image search, I found the specific place.

![](screenshots/Pasted%20image%2020260529151820.png)

Google identified the place as Modravský potok. Therefore, I focused on the village of Modrava. The next step was to find the specific route the photographer had taken.

My good old friend LLM identified one of the first photos as Luzenské údolí. Moreover, it mentioned a place called **Březník**, which later played a very important role in our investigation. We now had 2 places to start from.

![](screenshots/Pasted%20image%2020260529152806.png)

After entering the route **Modrava - Luzenské údolí** into mapy.com, we found this possible route which the author had probably taken. We can also notice that Březník is nearby.

![](screenshots/Pasted%20image%2020260529153101.png)

On the website [https://www.sumava-bavorskyles.cz/gabreta/fr.asp?tab=gabreta&id=752&burl=&pt=TT](https://www.sumava-bavorskyles.cz/gabreta/fr.asp?tab=gabreta&id=752&burl=&pt=TT), which provides hiking tips, I found the 5 most popular routes starting from Modrava. The only route passing through Březník / Luzenské údolí was **the circular route: Modrava – Březník – Cikánská slať – Modrava**.

![](screenshots/Pasted%20image%2020260529150942.png)

According to the task description, the author was returning after a late lunch and saw the signpost we were looking for on the way back at around 3 PM. This would correspond to approximately halfway through the **return journey**. Since the circular route mentions **Cikánská slať**, which happens to be almost exactly halfway between Modrava and Březník, it was worth trying to find a signpost at this location.

Thanks to the very helpful and passionate photographer Svatomír Rafaj, who photographed the signpost and published the photo, I was able to read the identification code on its sign, which was the flag we were looking for.

![](screenshots/Pasted%20image%2020260529150519.png)

![](screenshots/Pasted%20image%2020260529154843.png)

