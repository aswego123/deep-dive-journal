

🪝 The Hook
Labs spend hundreds of millions of dollars training a model BEFORE they know for sure it'll turn out good. How do they dare do that without knowing the outcome?

⚙️ Core Mechanism
Scaling laws are basically: "if I double the chefs and double the ingredients, does the banquet actually taste better — and can I predict by how much, before I cook it?"

Turns out yes. If you train a bunch of small, cheap models first, and plot how good they get as you give them more size and more data, the improvement follows a very smooth, predictable pattern. That pattern lets you draw the line forward and predict how good a MUCH bigger model will be — without actually building it yet. It's like taste-testing a mini version of a recipe to predict how the stadium-sized version will taste.

The important twist (discovered by a team at DeepMind, called the "Chinchilla" finding): for a long time, people were making models bigger (more dials) without feeding them proportionally more training text (ingredients). Turns out that's wasteful. The better move is to grow model size AND training data together, roughly evenly. A smaller, well-fed model can beat a bigger, underfed one — using the exact same amount of total compute.

🔗 Connects To

* [[AI And Compute]]
* [[What is Compute]]

💡 So What
When I read model release notes, I should now check: how big is it, AND how much data did it see? Big model + little data = probably wasted potential. That ratio actually matters more than raw size bragging rights.

❓ Open Question
Loss (a training score) follows this nice smooth predictable curve — but real skills like coding or math ability don't always follow it that smoothly, sometimes seeming to "suddenly appear." I don't yet understand why skill-level and loss-level don't always move together. Need to look into "emergent capabilities" debate.

📚 Source

* Claude conversation, July 13 2026 — kitchen analogy for scaling laws
* To read later: DeepMind's Chinchilla paper (2022), original OpenAI scaling laws paper (2020)
