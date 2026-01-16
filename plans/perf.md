Context: Hanayo Performance Optimization Project

Background

Working on performance improvements for akatsuki.gg (hanayo frontend) based on a Lighthouse report showing poor FCP (4.1s) and LCP (6.7s). We did a deep performance analysis that identified 16 categories of issues, then prioritized quick wins.

Completed PRs
┌───────────────────────┬──────────────────────────────────────────────────────────────────────────────┬───────────┐
│ PR │ Description │ Status │
├───────────────────────┼──────────────────────────────────────────────────────────────────────────────┼───────────┤
│ #35, #36 (nginx-conf) │ Improved gzip compression (gzip_types, level=6, vary, proxied) │ ✅ Merged │
├───────────────────────┼──────────────────────────────────────────────────────────────────────────────┼───────────┤
│ #200 (hanayo) │ Content hashing with gulp-rev + immutable caching │ ✅ Merged │
├───────────────────────┼──────────────────────────────────────────────────────────────────────────────┼───────────┤
│ #204 (hanayo) │ Scrollbar fix using flexbox sticky footer │ ✅ Merged │
├───────────────────────┼──────────────────────────────────────────────────────────────────────────────┼───────────┤
│ #205 (hanayo) │ Add defer to dist.min.js (~113KB render-blocking → non-blocking) │ ✅ Merged │
├───────────────────────┼──────────────────────────────────────────────────────────────────────────────┼───────────┤
│ #206 (hanayo) │ Fix page-specific scripts not using content hashes ({{ . }} → {{ asset . }}) │ ✅ Merged │
└───────────────────────┴──────────────────────────────────────────────────────────────────────────────┴───────────┘
Remaining Quick Wins (5 items)
┌─────┬──────────────────────────────────────────┬────────────────────┬──────────────────────────────────────────┐
│ # │ Optimization │ File │ Impact │
├─────┼──────────────────────────────────────────┼────────────────────┼──────────────────────────────────────────┤
│ 1 │ Add async to pace.js and twemoji scripts │ base.html:30,54 │ High - stops render blocking │
├─────┼──────────────────────────────────────────┼────────────────────┼──────────────────────────────────────────┤
│ 2 │ Change audio preload="auto" → "none" │ beatmap.html:35 │ Medium - saves 80-300KB per beatmap page │
├─────┼──────────────────────────────────────────┼────────────────────┼──────────────────────────────────────────┤
│ 3 │ Add loading="lazy" to flag images │ leaderboard.html │ Low - reduces initial network │
├─────┼──────────────────────────────────────────┼────────────────────┼──────────────────────────────────────────┤
│ 4 │ Add width/height to avatar images │ profile.html:70-73 │ High - fixes CLS (layout shift) │
├─────┼──────────────────────────────────────────┼────────────────────┼──────────────────────────────────────────┤
│ 5 │ Pin ApexCharts CDN version │ profile.js:20 │ Low - security/stability │
└─────┴──────────────────────────────────────────┴────────────────────┴──────────────────────────────────────────┘
Key Technical Context

Content Hashing System (PR #200):

- web/gulpfile.js generates hashed filenames via gulp-rev
- web/static/manifest.json maps original → hashed paths
- app/usecases/funcmap/funcmap.go has asset template function for lookups
- nginx serves /static/ with Cache-Control: public, max-age=31536000, immutable

Script Loading (PR #205):

- dist.min.js now has defer attribute
- Inline jQuery code wrapped in DOMContentLoaded listener since jQuery loads later with defer

Skipped Optimizations

- HTTP/2 in nginx - decided to skip
- ApexCharts lazy loading - already implemented in codebase
- Brotli compression - nginx:latest doesn't support it

Bigger Future Opportunities (from analysis)

- Extract unused Semantic CSS (could save 200-300KB)
- Bundle splitting (384KB dist.min.js → essential + interactive chunks)
- Migrate from Semantic UI 2.2.6 to lighter framework
