# Local build of the upstream image with ONE patch applied.
#
# Everything here mirrors InstagramEmbedForDiscord/Dockerfile from the pinned
# upstream commit, except the single `RUN` marked "patch".
#
# WHY: upstream hardcodes a 15s ceiling on the scrape it makes to its bundled
# snapsave service (CancellationTokenSource(TimeSpan.FromSeconds(15))). Measured
# against snapsave.app from the cluster, a successful scrape takes 1.4s-11s
# normally, with tails of 17s and 25s. Above the cap the scrape is abandoned and
# the request turns into a 404, so links that would have embedded fine show no
# embed at all. Raising the cap is the whole fix.

# ─── Stage 1 – Install snapsave Node dependencies ────────────────────────────
FROM node:22-alpine AS node-build

WORKDIR /snapsave
COPY snapsave/package.json ./
RUN npm install --omit=dev
COPY snapsave/index.js ./

# ─── Stage 2 – Build the .NET app (patched) ──────────────────────────────────
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS dotnet-build

WORKDIR /src
COPY InstagramEmbed.Application.csproj ./
RUN dotnet restore InstagramEmbed.Application.csproj
COPY . ./

# patch: raise both scrape ceilings, and FAIL THE BUILD if either upstream line
# moves - a silently unapplied patch would put us back to 404s.
#
#   1. PostCacheService's inner CancellationTokenSource cap (15s)
#   2. the named "snapsave" HttpClient timeout in Program.cs (20s)
#
# Measured against snapsave.app from this cluster: successful scrapes run
# 1.4s-11s when the upstream is healthy and 23s-41s when it is loaded. Either
# ceiling turns a working scrape into a 404. The inner token source is kept
# below the HttpClient timeout so cancellation reads as a clean timeout.
RUN set -eux; \
    f=Services/PostCacheService.cs; \
    grep -q 'TimeSpan.FromSeconds(15)' "$f"; \
    sed -i 's/TimeSpan\.FromSeconds(15)/TimeSpan.FromSeconds(60)/' "$f"; \
    grep -q 'TimeSpan.FromSeconds(60)' "$f"; \
    ! grep -q 'TimeSpan\.FromSeconds(15)' "$f"; \
    g=Program.cs; \
    grep -q 'c.Timeout = TimeSpan.FromSeconds(20)' "$g"; \
    sed -i 's/c\.Timeout = TimeSpan\.FromSeconds(20)/c.Timeout = TimeSpan.FromSeconds(120)/' "$g"; \
    grep -q 'c.Timeout = TimeSpan.FromSeconds(120)' "$g"; \
    ! grep -q 'c\.Timeout = TimeSpan\.FromSeconds(20)' "$g"

RUN dotnet publish InstagramEmbed.Application.csproj \
    -c Release \
    -o /app/publish \
    --no-restore

# ─── Stage 3 – Runtime ───────────────────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime

RUN apk add --no-cache nodejs

WORKDIR /app
COPY --from=dotnet-build /app/publish ./
RUN rm -rf ./snapsave
COPY --from=node-build /snapsave ./snapsave

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production
ENV SNAPSAVE_PORT=3200
ENTRYPOINT ["dotnet", "InstagramEmbed.Application.dll"]
