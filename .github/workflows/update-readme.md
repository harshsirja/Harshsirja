name: Auto-Update README Repositories

on:
  # Runs automatically every 6 hours to detect new repositories
  schedule:
    - cron: '0 */6 * * *'
  # Allows you to trigger the update manually from the "Actions" tab anytime
  workflow_dispatch:
  # Runs whenever you push changes to your profile repository
  push:
    branches:
      - main
      - master

permissions:
  contents: write

jobs:
  update-readme:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Profile Repository
        uses: actions/checkout@v4

      - name: Fetch Repositories & Update README.md
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const username = 'harshsirja';

            // Fetch all public repositories for harshsirja sorted by newest creation date
            const repos = await github.paginate(github.rest.repos.listForUser, {
              username: username,
              type: 'owner',
              sort: 'created',
              direction: 'desc',
              per_page: 100
            });

            // Filter out forks if you only want original repositories (set to true to include forks)
            const includeForks = false;
            const filteredRepos = repos.filter(repo => includeForks || !repo.fork);

            // Calculate aggregate metrics
            const totalRepos = filteredRepos.length;
            const totalStars = filteredRepos.reduce((acc, r) => acc + r.stargazers_count, 0);
            const totalForks = filteredRepos.reduce((acc, r) => acc + r.forks_count, 0);

            // Format date helper
            const formatDate = (dateString) => {
              const d = new Date(dateString);
              return d.toISOString().split('T')[0];
            };

            // Build summary badges and markdown table
            let markdownContent = `<!-- AUTO-REPO-LIST:START -->\n`;
            markdownContent += `<p align="center">\n`;
            markdownContent += `  <img src="https://img.shields.io/badge/Total_Public_Repos-${totalRepos}-58A6FF?style=flat-square&logo=github" alt="Total Repos" />\n`;
            markdownContent += `  <img src="https://img.shields.io/badge/Total_Stars-${totalStars}-f1e05a?style=flat-square&logo=star" alt="Total Stars" />\n`;
            markdownContent += `  <img src="https://img.shields.io/badge/Total_Forks-${totalForks}-238636?style=flat-square&logo=git" alt="Total Forks" />\n`;
            markdownContent += `</p>\n\n`;

            markdownContent += `| 📁 Repository | 📝 Description | 💻 Language | ⭐ Stars | 🍴 Forks | 📅 Created | 🕒 Updated |\n`;
            markdownContent += `| :--- | :--- | :---: | :---: | :---: | :---: | :---: |\n`;

            for (const repo of filteredRepos) {
              const name = `**[${repo.name}](${repo.html_url})**`;
              const desc = repo.description
                ? repo.description.replace(/\|/g, '\\|').replace(/\r?\n|\r/g, ' ')
                : '*No description provided*';
              const lang = repo.language ? `\`${repo.language}\`` : '`-`';
              const stars = repo.stargazers_count;
              const forks = repo.forks_count;
              const created = formatDate(repo.created_at);
              const updated = formatDate(repo.updated_at);

              markdownContent += `| ${name} | ${desc} | ${lang} | ${stars} | ${forks} | ${created} | ${updated} |\n`;
            }

            markdownContent += `<!-- AUTO-REPO-LIST:END -->`;

            // Read README.md and replace content between markers
            const readmePath = 'README.md';
            const readme = fs.readFileSync(readmePath, 'utf8');
            const regex = /<!-- AUTO-REPO-LIST:START -->[\s\S]*?<!-- AUTO-REPO-LIST:END -->/;

            if (!regex.test(readme)) {
              core.setFailed('Could not find <!-- AUTO-REPO-LIST:START --> and <!-- AUTO-REPO-LIST:END --> markers in README.md');
              return;
            }

            const updatedReadme = readme.replace(regex, markdownContent);

            if (updatedReadme !== readme) {
              fs.writeFileSync(readmePath, updatedReadme, 'utf8');
              console.log('README.md updated with latest repositories!');
            } else {
              console.log('No repository changes detected.');
            }

      - name: Commit and Push Changes
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git add README.md
          git diff --staged --quiet || git commit -m "chore(readme): auto-update repository overview list [skip ci]"
          git push
