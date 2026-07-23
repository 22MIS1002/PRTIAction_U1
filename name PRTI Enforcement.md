**name: PRTI Enforcement**



on:

&#x20; pull\_request:

&#x20;   types: \[opened, edited, reopened, synchronize]

&#x20; issue\_comment:

&#x20;   types: \[created, edited]



permissions:

&#x20; pull-requests: write

&#x20; issues: write

&#x20; statuses: write



jobs:

&#x20; prti-check:

&#x20;   if: >-

&#x20;     github.event\_name == 'pull\_request' ||

&#x20;     (github.event\_name == 'issue\_comment' \&\& github.event.issue.pull\_request)

&#x20;   runs-on: ubuntu-latest



&#x20;   steps:

&#x20;     - name: Validate PRTI data, update sticky comment, set commit status

&#x20;       uses: actions/github-script@v7

&#x20;       with:

&#x20;         script: |

&#x20;           const MARKER = '<!-- prti-reminder-sticky -->';

&#x20;           const STATUS\_CONTEXT = 'prti-check';

&#x20;           const { owner, repo } = context.repo;



&#x20;           // Prevent infinite loop if comment is created by GitHub Actions bot

&#x20;           if (context.eventName === 'issue\_comment' \&\& context.payload.comment.user.login === 'github-actions\[bot]') {

&#x20;             core.info('Comment created by GitHub Actions bot. Skipping execution.');

&#x20;             return;

&#x20;           }



&#x20;           const prNumber = context.eventName === 'issue\_comment'

&#x20;             ? context.payload.issue.number

&#x20;             : context.payload.pull\_request.number;



&#x20;           const { data: pr } = await github.rest.pulls.get({

&#x20;             owner, repo, pull\_number: prNumber

&#x20;           });



&#x20;           const headSha = pr.head.sha;

&#x20;           const author = pr.user ? pr.user.login : 'there';



&#x20;           // Fetch all comments on the PR

&#x20;           let comments = await github.paginate(

&#x20;             github.rest.issues.listComments,

&#x20;             { owner, repo, issue\_number: prNumber, per\_page: 100 }

&#x20;           );



&#x20;           const isSynchronize = context.eventName === 'pull\_request' \&\& context.payload.action === 'synchronize';

&#x20;           const prtiRegex = /PR\\s\*Time|PR\\s\*Defects|TI\\s\*Time/i;



&#x20;           // 1. Delete old PRTI comments on new commits

&#x20;           if (isSynchronize) {

&#x20;             core.info('New commit(s) pushed. Cleaning up obsolete PRTI comments...');

&#x20;             const prtiCommentsToDelete = comments.filter(c => 

&#x20;               c.user \&\& 

&#x20;               c.user.login === author \&\& 

&#x20;               !c.body.includes(MARKER) \&\& 

&#x20;               prtiRegex.test(c.body)

&#x20;             );



&#x20;             for (const comment of prtiCommentsToDelete) {

&#x20;               await github.rest.issues.deleteComment({

&#x20;                 owner, repo, comment\_id: comment.id

&#x20;               });

&#x20;               core.info(`Deleted obsolete PRTI comment ID: ${comment.id}`);

&#x20;             }



&#x20;             comments = await github.paginate(

&#x20;               github.rest.issues.listComments,

&#x20;               { owner, repo, issue\_number: prNumber, per\_page: 100 }

&#x20;             );

&#x20;           }



&#x20;           // 2. Scan PR Description \& author comments (reversed to get newest comments first)

&#x20;           const authorComments = comments

&#x20;             .filter(c => c.user \&\& c.user.login === author)

&#x20;             .filter(c => !(c.body || '').includes(MARKER))

&#x20;             .map(c => c.body || '');



&#x20;           // Order sources: Newest comment first, falling back to PR description

&#x20;           const textSources = \[...authorComments.reverse(), pr.body || ''];



&#x20;           // Helper function to extract the most recent value for a given field regex

&#x20;           function getLatestValue(regex) {

&#x20;             for (const text of textSources) {

&#x20;               const match = text.match(regex);

&#x20;               if (match \&\& match\[1] !== undefined) {

&#x20;                 return match\[1].trim();

&#x20;               }

&#x20;             }

&#x20;             return null;

&#x20;           }



&#x20;           // 3. Validation Logic Engine

&#x20;           const checks = \[

&#x20;             { label: 'PR Time',    re: /PR\\s\*Time\\s\*:\\s\*(.\*)$/im,    type: 'time' },

&#x20;             { label: 'PR Defects', re: /PR\\s\*Defects\\s\*:\\s\*(.\*)$/im, type: 'number' },

&#x20;             { label: 'TI Time',    re: /TI\\s\*Time\\s\*:\\s\*(.\*)$/im,    type: 'time' },

&#x20;           ];



&#x20;           const invalidFields = \[];



&#x20;           for (const c of checks) {

&#x20;             const rawValue = getLatestValue(c.re);



&#x20;             // Check 1: Empty or whitespace only

&#x20;             if (rawValue === null || rawValue === '') {

&#x20;               invalidFields.push(`${c.label} (Empty/Missing)`);

&#x20;               continue;

&#x20;             }



&#x20;             // Check 2: Reject placeholders (e.g. <enter time...>, N/A, TBD, TODO)

&#x20;             if (/^<.\*>$/i.test(rawValue) || /^(n\\/a|na|tbd|todo|\\?+)$/i.test(rawValue)) {

&#x20;               invalidFields.push(`${c.label} (Invalid placeholder: "${rawValue}")`);

&#x20;               continue;

&#x20;             }



&#x20;             // Check 3: Reject negative numbers

&#x20;             if (rawValue.includes('-')) {

&#x20;               invalidFields.push(`${c.label} (Cannot be negative: "${rawValue}")`);

&#x20;               continue;

&#x20;             }



&#x20;             // Check 4: Type-specific validation

&#x20;             if (c.type === 'number') {

&#x20;               // Must strictly be a non-negative integer (e.g. 0, 1, 2)

&#x20;               if (!/^\\d+$/.test(rawValue)) {

&#x20;                 invalidFields.push(`${c.label} (Must be a non-negative number, e.g., 0)`);

&#x20;               }

&#x20;             } else if (c.type === 'time') {

&#x20;               // Time must contain at least one digit (e.g., 2h, 1.5h, 30m, 0)

&#x20;               if (!/\\d/.test(rawValue)) {

&#x20;                 invalidFields.push(`${c.label} (Must contain a valid time duration, e.g., 2h or 30m)`);

&#x20;               }

&#x20;             }

&#x20;           }



&#x20;           const allValid = invalidFields.length === 0;



&#x20;           // 4. Build status message and contextual banners

&#x20;           let bannerLines = \[];



&#x20;           if (isSynchronize) {

&#x20;             bannerLines = \[

&#x20;               '> 🔄 \*\*New Commits Detected!\*\*',

&#x20;               '> Previous PRTI data has been invalidated. Please submit updated PRTI details for this new revision.',

&#x20;               ''

&#x20;             ];

&#x20;           } else if (context.eventName === 'issue\_comment' \&\& !allValid) {

&#x20;             const latestCommentBody = context.payload.comment.body || '';

&#x20;             if (prtiRegex.test(latestCommentBody)) {

&#x20;               bannerLines = \[

&#x20;                 '> ⚠️ \*\*Notice on your recent comment:\*\*',

&#x20;                 '> We detected a PRTI submission attempt, but some fields failed validation.',

&#x20;                 ''

&#x20;               ];

&#x20;             }

&#x20;           }



&#x20;           let statusLines;

&#x20;           if (allValid) {

&#x20;             statusLines = \[

&#x20;               '### PRTI Checklist Complete',

&#x20;               'All required PRTI fields were detected and validated for the current revision. Thank you!'

&#x20;             ];

&#x20;           } else {

&#x20;             statusLines = \[

&#x20;               '### PRTI Checklist Incomplete or Invalid',

&#x20;               'The following issue(s) need attention:',

&#x20;               ...invalidFields.map(err => `- \*\*${err}\*\*`),

&#x20;               '',

&#x20;               'This PR is \*\*blocked from merging\*\* until valid PRTI data is supplied.'

&#x20;             ];

&#x20;           }



&#x20;           // 5. Assemble full sticky comment body

&#x20;           const bodyLines = \[

&#x20;             MARKER,

&#x20;             '## PRTI Checklist Enforcement',

&#x20;             '',

&#x20;             ...bannerLines,

&#x20;             'Hi @' + author + ', please provide your \*\*PRTI details\*\* in the PR description or a comment.',

&#x20;             '',

&#x20;             '### Required Format (Copy \& Paste):',

&#x20;             '```text',

&#x20;             'PR Time    : <enter time, e.g., 2h>',

&#x20;             'PR Defects : <enter defects count, e.g., 0>',

&#x20;             'TI Time    : <enter time, e.g., 1h>',

&#x20;             '```',

&#x20;             '',

&#x20;             '| Field | Validation Rule | Meaning |',

&#x20;             '|-------|-----------------|---------|',

&#x20;             '| \*\*PR Time\*\* | Time duration (e.g. `2h`, `30m`) | Time spent creating this PR |',

&#x20;             '| \*\*PR Defects\*\* | Non-negative number (e.g. `0`, `1`) | Defects found during review |',

&#x20;             '| \*\*TI Time\*\* | Time duration (e.g. `1h`, `45m`) | Time spent on Technical Investigation |',

&#x20;             '',

&#x20;             '---',

&#x20;             ...statusLines,

&#x20;             '',

&#x20;             '> Automated by the PRTI Enforcement workflow. Re-evaluates on new commits.'

&#x20;           ];



&#x20;           const body = bodyLines.join('\\n');



&#x20;           // 6. Update or create the sticky comment

&#x20;           const existingSticky = comments.find(c => (c.body || '').includes(MARKER));



&#x20;           if (existingSticky) {

&#x20;             await github.rest.issues.updateComment({

&#x20;               owner, repo, comment\_id: existingSticky.id, body

&#x20;             });

&#x20;           } else {

&#x20;             await github.rest.issues.createComment({

&#x20;               owner, repo, issue\_number: prNumber, body

&#x20;             });

&#x20;           }



&#x20;           // 7. Update Commit Status

&#x20;           const description = allValid

&#x20;             ? 'PRTI checklist complete'

&#x20;             : ('Validation failed: ' + invalidFields.join(', ')).slice(0, 140);



&#x20;           await github.rest.repos.createCommitStatus({

&#x20;             owner,

&#x20;             repo,

&#x20;             sha: headSha,

&#x20;             context: STATUS\_CONTEXT,

&#x20;             state: allValid ? 'success' : 'failure',

&#x20;             description,

&#x20;             target\_url: 'https://github.com/' + owner + '/' + repo + '/pull/' + prNumber

&#x20;           });



&#x20;           if (allValid) {

&#x20;             core.info('PRTI checklist complete - commit status set to success.');

&#x20;           } else {

&#x20;             core.warning('PRTI checklist incomplete or invalid. Issues: ' + invalidFields.join(', '));

&#x20;           }



