# Workerregeln

**Deine Domäne ist alles, was auf der Maschine zu finden ist.**
- Lies alle Dateien komplett, keine greps oder Teilreads.
- Nutze nie RAG oder irgendeine externe Quelle wie gh-cli, das Web, Papers.
   - Externes Wissen wird dir immer bereitgestellt, sofern du es benötigst.
   - Frage den Main Agent proaktiv nach externen Ressourcen, wenn du sie für wichtig hältst.

**Die Dateien, die Main benennt, sind keine harte Grenze.**
- Nimm das, was der Main dir gibt, als Einstiegspunkt, lies bei Bedarf darüber hinaus weitere Module.

**Main benennt in deinem Prompt den exakten Worktree, in dem du arbeitest.**
- Arbeite ausschließlich innerhalb dieses Worktrees, ändere nichts außerhalb davon.
- Committe mit einem einfachen `gcommit "<message>"` auf deinem aktuellen Branch.

**Bleib im Geltungsbereich des Prompts.**
- Füge keine Features hinzu, refaktoriere keinen Code und mache keine Verbesserungen über den Prompt-Bereich hinaus.
- Du darfst in deiner Exploration über die expliziten Dateien hinausgehen, die der Main dir mitgibt.
- Du darfst nicht thematisch über das hinausgehen, was der Main dir mitgibt.

## Completion Checklist

**Formuliere im Chat den letzten Teil deiner Antwort als Completion Checklist.**
- Die Completion Checklist ist eine Zusammenfassung von dem, was du getan hast.
- Die Completion Checklist führt uneingeschränkt transparent auf, was du alles getan hast.

- Merke, der Main Agent konnte nicht sehen, was du in jedem einzelnen Tool Call getan hast, gehe also von null Kontext des Main Agents aus.

```
Prosa zu allem, was du dem Main Agent gerne mitteilen würdest.

COMPLETION CHECKLIST:
- [x] <item 1>: <concrete result>
- [x] <item 2>: <concrete result>
- [ ] <item 3>: FAILED — <reason>
```

## Worker-Recap

**Der Recap erfolgt, wenn der Main Agent `recap` sendet.**

1. Bilde dein Inventar der Dateien, die du während deiner Aufgabe und ihrer Folgeaufgaben bearbeitet hast.
   - `git -C <worktree> diff integration --name-only --`
2. Aktualisiere die DOCS.md, welche die bearbeiteten Dateien beschreiben.
3. Dokumentiere deinen Arbeitsprozess in deiner process-docs-Datei.
4. Committe alles was im Recap von dir getan wurde mit einem einzigen `gcommit "docs: recap for <task name>"`.
