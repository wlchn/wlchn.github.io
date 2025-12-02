# Lei Wang's Blog

This is a personal blog built with Hugo and the PaperMod theme.

## Features

- Built with Hugo 0.152.2
- PaperMod theme
- GitHub Pages deployment
- Responsive design
- Search functionality
- Archive page

## Local Development

```bash
# Install Hugo (macOS with Homebrew)
brew install hugo

# Clone the repository
git clone https://github.com/wlchn/wlchn.github.io.git
cd wlchn.github.io

# Initialize submodules
git submodule update --init --recursive

# Start development server
hugo server --buildDrafts

# Build for production
hugo --gc --minify
```

## Deployment

This site is automatically deployed to GitHub Pages using GitHub Actions when changes are pushed to the main branch.

## Content

All blog posts are located in the `content/posts/` directory and are written in Markdown format.

## License

This project is open source and available under the [MIT License](LICENSE).
