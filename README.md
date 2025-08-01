# DevLog & Homelab Chronicles

A technical blog documenting homelab adventures, cloud certifications, and real-world problem-solving in the trenches of self-hosted infrastructure.

## 🚀 Quick Start

This blog is built with Jekyll and designed to run seamlessly on GitHub Pages.

### Local Development

1. **Install dependencies:**
   ```bash
   bundle install
   ```

2. **Run the development server:**
   ```bash
   bundle exec jekyll serve
   ```

3. **Visit your site:**
   Open [http://localhost:4000](http://localhost:4000) in your browser

### GitHub Pages Deployment

1. **Enable GitHub Pages:**
   - Go to your repository settings
   - Navigate to "Pages" section
   - Set source to "GitHub Actions"

2. **Push to main branch:**
   ```bash
   git add .
   git commit -m "Initial blog setup"
   git push origin main
   ```

3. **Automatic deployment:**
   - GitHub Actions will automatically build and deploy your site
   - Your blog will be available at `https://njvanas.github.io/Blog`

## 📝 Writing Posts

### Create a New Post

1. **Create a new file** in the `_posts` directory:
   ```
   _posts/YYYY-MM-DD-your-post-title.md
   ```

2. **Use the front matter template:**
   ```yaml
   ---
   title: "Your Post Title"
   date: YYYY-MM-DD
   tags: [homelab, devlog, troubleshooting, docker]
   layout: post
   ---
   ```

3. **Follow the structured format:**
   - 📘 Overview
   - 🧱 Project/Setup Context  
   - 💥 Roadblocks & Shortcomings
   - 🧠 How I Resolved the Issues
   - 📚 What I Learned
   - 🎯 Outcome & Next Steps
   - 🧭 Related Resources/Tools
   - 💬 Final Thoughts

### Supported Tags

Use these tags to categorize your posts:
- `homelab` - Self-hosted infrastructure projects
- `devlog` - Development and coding projects
- `exam-prep` - Certification study notes
- `selfhosted` - Self-hosting specific content
- `troubleshooting` - Problem-solving focused posts
- `docker` - Container-related content
- `azure` - Microsoft Azure content
- `aws` - Amazon Web Services content
- `networking` - Network configuration and issues
- `security` - Security-focused content

## 🎨 Customization

### Site Configuration

Edit `_config.yml` to customize:
- Site title and description
- Author information
- Social media links
- URL structure

### Styling

- **Main styles:** `assets/custom.css`
- **Color scheme:** Modify CSS variables in custom.css
- **Layout:** Edit files in `_layouts/` directory

### Features

- **Syntax highlighting** with Rouge
- **Responsive design** for mobile and desktop
- **SEO optimization** with jekyll-seo-tag
- **RSS feed** generation
- **Archive page** with posts grouped by year
- **Tag system** for content organization

## 📁 Project Structure

```
├── _config.yml          # Site configuration
├── _posts/              # Blog posts
├── _layouts/            # Page templates
├── _includes/           # Reusable components
├── assets/              # CSS, images, etc.
├── .github/workflows/   # GitHub Actions
├── index.md             # Homepage
├── archive.md           # Archive page
├── Gemfile              # Ruby dependencies
└── README.md            # This file
```

## 🛠️ Advanced Features

### Code Blocks

Use fenced code blocks with syntax highlighting:

```yaml
# Example YAML
services:
  web:
    image: nginx
    ports:
      - "80:80"
```

### Alert Boxes

Add custom alert boxes in your posts:

```html
<div class="alert alert-info">
💡 <strong>Pro Tip:</strong> This is an informational alert.
</div>
```

### Tables

Create responsive tables:

| Feature | Status | Notes |
|---------|--------|-------|
| SSL | ✅ | Automated via Let's Encrypt |
| Backup | ✅ | Daily automated backups |
| Monitoring | 🔄 | In progress |

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test locally
5. Submit a pull request

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

---

**Happy blogging!** 🎉

For questions or suggestions, feel free to open an issue or reach out via the contact methods listed on the blog.