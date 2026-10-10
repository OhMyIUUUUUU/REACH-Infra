FROM golang:1.27-alpine AS build

WORKDIR /src
COPY reach/Backend/go.mod ./
COPY reach/Backend/*.go ./
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/reach-backend .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/reach-backend /reach-backend
ENV PORT=8080
EXPOSE 8080
ENTRYPOINT ["/reach-backend"]
