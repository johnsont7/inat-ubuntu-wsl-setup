```markdown
# iNaturalist Development Setup Guide for Ubuntu on WSL

This guide walks through setting up the iNaturalist Rails web application and Node.js API on Ubuntu running inside WSL 2, including all required system dependencies, optional GIS/admin setup, and fixes for common compilation and environment issues.

---

## Prerequisites

- **Windows with WSL 2** running Ubuntu (20.04, 22.04, or 24.04+).
- **Docker Desktop for Windows** running with **WSL 2 Integration** enabled for your Ubuntu distribution (Settings > Resources > WSL Integration), or Docker Engine installed directly in Ubuntu.

---

## 1. Install System Dependencies & Build Tools

Update your package list and install required system libraries, build tools, database headers, GIS tools, and media processors:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
  git \
  curl \
  pkg-config \
  build-essential \
  libssl-dev \
  libreadline-dev \
  zlib1g-dev \
  imagemagick \
  exiftool \
  ffmpeg \
  libpq-dev \
  libgeos-dev \
  libgdal-dev \
  gdal-bin \
  unzip

```

---

## 2. Install RVM and NVM

### RVM (Ruby Version Manager)

Import both maintainer GPG keys (including Piotr Kuczynski's key to avoid signature verification failures) and install RVM:

```bash
curl -sSL [https://rvm.io/mpapis.asc](https://rvm.io/mpapis.asc) | gpg --import -
curl -sSL [https://rvm.io/pkuczynski.asc](https://rvm.io/pkuczynski.asc) | gpg --import -
\curl -sSL [https://get.rvm.io](https://get.rvm.io) | bash -s stable
source ~/.rvm/scripts/rvm

```

Persist RVM in future terminal sessions:

```bash
echo 'source ~/.rvm/scripts/rvm' >> ~/.bashrc
source ~/.bashrc

```

Verify RVM is loaded:

```bash
rvm -v

```

### NVM (Node Version Manager)

```bash
curl -o- [https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh](https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh) | bash
export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" \vert{}\vert{} printf \%s "${XDG_CONFIG_HOME}")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

```

---

## 3. Clone & Configure the Rails App

Clone both repositories into sibling directories:

```bash
git clone [https://github.com/inaturalist/inaturalist.git](https://github.com/inaturalist/inaturalist.git)
git clone [https://github.com/inaturalist/iNaturalistAPI.git](https://github.com/inaturalist/iNaturalistAPI.git)

cd inaturalist

```

Install the exact Ruby version required by the repository and install Bundler:

```bash
rvm install $(cat .ruby-version)
rvm use $(cat .ruby-version) --default
gem install bundler

```

### Troubleshooting Native Gem Compilation (`ed25519` & C Compiler Errors)

Modern Ubuntu releases ship with newer GCC versions that treat older C patterns in native Ruby gems as fatal errors.

1. **Fix `ed25519` (1.3.0) build failure:** Open `Gemfile`, locate the `:development` group, and constrain `ed25519` to `~> 1.4`:
```ruby
group :development do
  # The following are required for ed25519 ssh keys
  gem "bcrypt_pbkdf"
  gem "ed25519", "~> 1.4"
end

```


2. **Optional C compiler flags (if other native gems like `pg` fail to build):**
```bash
export CFLAGS="-Wno-error=implicit-function-declaration -Wno-error=int-conversion -Wno-error=incompatible-pointer-types"

```



Install all Ruby gem dependencies:

```bash
bundle install

```

---

## 4. Start Docker Services & Initialize the Database

Copy the example Docker Compose override file:

```bash
cp docker-compose.override.yml.example docker-compose.override.yml

```

*(Customize database credentials in `docker-compose.override.yml` if you do not want the default `inaturalist` / `inaturalist` Postgres user. If you hit an architecture manifest error for Postgres, add `platform: linux/amd64` under the `pg` service.)*

Start the containerized services (PostgreSQL, Elasticsearch, Redis), set up the database, and launch the Rails server:

```bash
# Start background services
make services

# Create and migrate the database
ruby bin/setup

# Start the Rails server
bundle exec rails s

```

Verify the app loads at `http://localhost:3000`.

---

## 5. Build the Frontend Assets (Webpack)

Open a new terminal tab inside the `inaturalist` directory:

```bash
nvm install
nvm use
npm install
npm run webpack

```

*(Tip: Use `npm run webpack-watch` during active development to automatically rebuild React files on save.)*

---

## 6. Setup & Run the Node.js API

Core pages (such as observation and taxon detail views) depend on the Node API running alongside Rails. Open a new terminal tab and navigate to `iNaturalistAPI`:

```bash
cd ../iNaturalistAPI
nvm install
nvm use
cp config_example.js config.js
npm install
node app.js

```

Verify the API is running at `http://localhost:4000`, and test `http://localhost:3000/taxa/Animalia` in your browser.

---

## 7. Optional Post-Setup Configuration

Run these commands from the `inaturalist` directory as needed:

### Create an Admin User

```bash
bundle exec rails r "admin = User.create!( login: 'admin', email: 'admin@youremaildomain.net', password: 'adminpassword', password_confirmation: 'adminpassword', confirmed_at: Time.now ); admin.roles << Role::ADMIN_ROLE"

```

### Load Timezones, Places & Background Jobs

```bash
# Download and load timezone boundaries
curl -OL "[https://github.com/evansiroky/timezone-boundary-builder/releases/download/2021c/timezones-with-oceans.shapefile.zip](https://github.com/evansiroky/timezone-boundary-builder/releases/download/2021c/timezones-with-oceans.shapefile.zip)"
unzip timezones-with-oceans.shapefile.zip
bundle exec rails r tools/load_time_zone_geometries.rb combined-shapefile-with-oceans.shp

# Import countries, states, and counties
bundle exec rails r tools/import_natural_earth_countries.rb
bundle exec rails r tools/import_us_states.rb
bundle exec rails r tools/import_us_counties.rb # Optional; takes a long time

# Start background worker for dashboard notifications
ruby script/delayed_job start

```

---

## 8. Running Tests

```bash
# Rails (single spec example — full suite takes a very long time)
bundle exec rspec spec/models/saved_location_spec.rb

# Node API (from iNaturalistAPI directory)
npm test

```

```

```