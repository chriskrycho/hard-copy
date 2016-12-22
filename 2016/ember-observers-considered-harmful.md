---
Title: Ember Observers considered harmful.
Subtitle: Or, well, worth using very carefully at least.
Date: 2016-10-14 14:00
Status: draft
---

> Observers are often over-used by new Ember developers. Observers are used heavily within the Ember framework itself, but for most problems Ember app developers face, computed properties are the appropriate solution.

In the last week, I found myself in a spot with the Ember app I work on where I'd previously used Ember's *observers*, writing something like `foo: observer('model.quantity', function() { … })`. And I found myself in deep, *deep* pain.

