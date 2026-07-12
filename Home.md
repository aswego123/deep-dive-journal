---
tags: [moc]
---

# 🧠 Deep Dive Journal

Welcome to your reach/reading journal. This is home base start here.

```dataviewjs
dv.span("**📚 Deep Drive Streak**")

const calendarData = {
	year: 2026,
	colors: {
		green: ["#c6e48b", "#7bc96f", "#49af5d", "#2e8840", "#196127"],
	},
	showCurrentDayBorder: true,
	defaultEntryIntensity: 4,
	intensityScaleStart: 1,
	intensityScaleEnd: 4,
	entries: [],
}

for (let page of dv.pages('"DeepDiveJournal/Daily Notes"').where(p => p.journaled)) {
	calendarData.entries.push({
		date: page.file.name,
		intensity: page.journaled,
		content: "",
		color: "green",
	})
}

renderHeatmapCalendar(this.container, calendarData)
```

## Current Season
> Update this each time you rotate themes (2–3 weeks per season)

**Season:** _e.g. Mind & Behavior_
**Dates:** _____ to _____
**Active topic:** [[Neuroplasticity]]

## Clusters (this vault's top-level tree)
- [[Mind & Behavior]]
- [[Thinking Tools]]
- [[Decision & Power]]
- [[Philosophy of Mind & Self]]
- [[Frontier Science]] 

## Quick Links
- [[Templates/Topic Entry Template]]
- [[Templates/Weekly Review Template]]
- 📁 All entries live under their Cluster note in `Clusters/`
- 📁 All weekly syntheses live in `Weekly Reviews/`

## How the tree works
Home → Cluster → Topic → Sub-branch
