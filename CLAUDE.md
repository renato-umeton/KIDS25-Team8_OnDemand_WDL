# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Sprocket OnDemand Application** - a web-based GUI for launching WDL (Workflow Description Language) pipelines on LSF HPC clusters. Users submit bioinformatics workflows through a visual interface instead of command-line tools.

**Key Context**: This is an Open OnDemand (OOD) interactive application, not a traditional software project. It consists primarily of configuration templates and form definitions.

## Architecture

The application follows OnDemand's template-driven architecture:

### Core Components
- **`form.yml.erb`**: Defines the web form users interact with (workflow selection, resource requirements)
- **`submit.yml.erb`**: Configures how jobs are submitted to LSF
- **`template/script.sh.erb`**: Bash script executed on compute nodes that runs Sprocket
- **`template/sprocket.toml.erb`**: Runtime configuration for Sprocket's LSF+Apptainer backend
- **`manifest.yml`**: Application metadata for OnDemand
- **`form.js`**: Vue.js frontend for form interactivity (96KB minified bundle)

### Configuration Flow
1. User fills web form → `form.yml.erb` processes input
2. Form data populates ERB templates → generates job submission config
3. LSF job launches → executes `script.sh.erb` on compute node
4. Script loads Sprocket module → runs WDL pipeline with generated config

### Resource Architecture
- **Sprocket orchestrator**: Runs with modest resources (1 core, 2GB RAM)
- **WDL tasks**: Get their own resource specifications from workflow definitions
- **Working directory**: `/scratch_space/$USER/sprocket`

## Development Workflow

### No Traditional Build System
This is a configuration-driven application - changes are made by editing templates and configuration files.

### Common Development Tasks

#### Testing Changes
```bash
# After modifying templates, test by:
# 1. Deploy to OnDemand development instance
# 2. Submit test workflow through web interface
# 3. Monitor execution via OnDemand logs
# 4. Check `/scratch_space/$USER/sprocket/output.log`
```

#### Key Configuration Points
- **`form.yml.erb:4`**: Cluster-specific CPU queue configuration via `OodAppkit.clusters[:hpc].custom_config[:cpu_queues]`
- **`form.yml.erb:13`**: Sprocket module version (currently `sprocket/0.17.1+520`)
- **`template/sprocket.toml.erb:3`**: Backend type (`ondemand` with experimental features)
- **`template/script.sh.erb:22`**: Module loading command

#### Deployment
```bash
# Deploy to OnDemand apps directory
cp -r . /var/www/ood/apps/sys/sprocket/

# Or link for development
ln -s $(pwd) /var/www/ood/apps/sys/sprocket
```

### Form Field Dependencies

ERB context variables available in all templates:
- `context.workflow` - WDL workflow file path/URL
- `context.inputs` - JSON/YAML inputs or command-line arguments
- `context.entrypoint` - Specific workflow/task to run
- `context.config` - User-provided Sprocket config file
- `context.queue` - Selected LSF queue
- `context.modules` - Module string to load
- `context.max_scatter_concurrency` - Concurrent scatter tasks limit
- `context.apptainer_images_dir` - Container cache directory

**Critical**: Form field names in `form.yml.erb` must exactly match template variable names.

## Technical Requirements

### Environment Dependencies
- **Open OnDemand**: Web server with LSF submission capability
- **LSF**: Active cluster with queues defined in cluster config
- **Modules**: Sprocket module available (`module load sprocket/0.17.1+520`)
- **Apptainer**: Container runtime for workflow execution
- **Scratch Space**: `/scratch_space/$USER/sprocket` must be writable

### Cluster Configuration
Requires OnDemand cluster config at `/etc/ood/config/clusters.d/*.yml` with:
```yaml
custom_config:
  cpu_queues:
    - "normal"
    - "priority"
    - "long"
```

## Common Issues and Patterns

### Recent Fixes (from git history)
- **Config file handling**: Support both user-provided and generated configs
- **Memory allocation**: Sprocket job memory bumped to 2048 MiB (`template/script.sh.erb`)
- **Cache directory**: Remove incorrect default for Apptainer images directory
- **Module versions**: Keep Sprocket module version updated

### ERB Template Debugging
```erb
# Check if variable exists before using
<%- unless context.config.blank? -%>
-c <%= context.config %>
<%- end -%>

# Loop through cluster configuration
<%- cpu_queues.each do |q| -%>
  - [ "<%= q %>", "<%= q %>" ]
<%- end -%>
```

### LSF Job Naming Convention
Pattern: `sprocket<cores>CPU_<mem>MEM_<hours>hr`
- Example: `sprocket1CPU_2048MEM_24hr`

## File Structure Conventions

```
├── manifest.yml                 # App metadata (version, description)
├── form.yml.erb                 # Form definition with ERB templating
├── form.js                      # Vue.js frontend (don't modify directly)
├── submit.yml.erb               # LSF submission configuration
├── template/
│   ├── script.sh.erb           # Main execution script
│   └── sprocket.toml.erb       # Sprocket runtime config
├── view.html.erb               # User guidance page
└── icon.png                    # OnDemand interface icon
```

## Testing Strategy

Manual testing through OnDemand interface:
1. Submit test workflow with minimal resource requirements
2. Monitor job submission to LSF queue
3. Check execution logs in `/scratch_space/$USER/sprocket/`
4. Verify Sprocket can pull containers and execute WDL tasks
5. Test different input formats (JSON files vs command-line arguments)

No automated testing framework - this is tested via integration with actual HPC environment.