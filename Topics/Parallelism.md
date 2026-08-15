

🪝 The Hook
If one chip would take hundreds of years to train a big model, and you have thousands of chips instead how do you actually SPLIT the work so the chips don't just trip over each other?

⚙️ Core Mechanism
This is the "organizing thousands of chefs in one kitchen" problem. There are three main ways to split the work:

1. Data parallel — every chef gets the FULL recipe book (a full copy of the model), but a different slice of guests to cook for (different training data). Simple, but only works if the recipe book (model) is small enough to fit in front of every single chef.

2. Tensor parallel — the recipe book itself is too big for one counter (one chip doesn't have enough memory), so you literally tear out pages and give different chefs different pages of the SAME dish. They have to talk to each other constantly to combine their pieces, so this only works well between chefs standing right next to each other (chips in the same server).

3. Pipeline parallel — assembly-line style. Chef 1 makes the appetizer, hands it to chef 2 for the main course, hands it to chef 3 for dessert. Less constant talking needed, so this works OK even between chefs (chips) further apart.

Real big training runs use all three at once, layered to match how "close" chips physically are to each other — tensor parallel for chips sitting right next to each other, pipeline parallel for chips a bit further apart, data parallel for whole separate groups far apart.

There's also a wiring problem: how fast can chips actually pass work to each other? This is called the "interconnect." Doesn't matter how fast a chef chops vegetables if the window to pass the dish to the next station is slow — a cluster with slightly weaker chips but great wiring can beat a cluster with stronger chips and bad wiring.

🔗 Connects To

* [[AI And Compute]]
* [[What is Compute]]

💡 So What
When someone brags about "we have 10,000 GPUs," I now know that number alone doesn't tell the full story — HOW those GPUs are wired together matters just as much as how many there are.

❓ Open Question
I still don't fully picture what a "pipeline bubble" (idle chip time) looks like in practice, or how micro-batches actually fix it. Need a visual/diagram to really get this one.

📚 Source

* Claude conversation, July 13 2026 — kitchen analogy for parallelism
