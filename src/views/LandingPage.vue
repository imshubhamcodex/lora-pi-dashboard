<template>
  <v-card class="overflow-hidden">
    <v-app-bar
      absolute
      color="#43a047"
      dark
      shrink-on-scroll
      prominent
      src="https://picsum.photos/1920/1080?random"
      fade-img-on-scroll
      scroll-target="#scrolling-techniques-5"
      scroll-threshold="300"
    >
      <template v-slot:img="{ props }">
        <v-img
          v-bind="props"
          gradient="to top right, rgba(55,236,186,.7), rgba(25,32,72,.7)"
        ></v-img>
      </template>

      <p class="text-h4 mt-2 mb-1">
        <v-icon size="100" class="mr-5">mdi-forest</v-icon> Plant Disease
        Detection Dashboard
      </p>

      <v-spacer></v-spacer>

      <v-btn icon>
        <v-icon>mdi-calendar</v-icon>
      </v-btn>

      <v-btn icon>
        <v-icon>mdi-filter-settings</v-icon>
      </v-btn>

      <v-btn icon>
        <v-icon>mdi-image-filter-center-focus-strong</v-icon>
      </v-btn>
    </v-app-bar>
    <v-sheet id="scrolling-techniques-5" class="overflow-y-auto">
      <v-container style="margin-top: 220px">
        <v-row>
          <v-col cols="auto">
            <v-data-table
              class="elevation-1"
              :headers="headers"
              :items="packets"
              dense
            />
          </v-col>
        </v-row>
        <v-row justify="center" style="margin-top: 20px">
          <v-col cols="auto" style="position: relative">
            <canvas
              ref="gridCanvas"
              :width="h_w"
              :height="h_w"
              style="border: 1px solid #ccc"
              @click="onClickCell"
            ></canvas>
            <div
              v-if="tooltip"
              :style="{
                position: 'absolute',
                top: tooltipY + 'px',
                left: tooltipX + 'px',
                background: 'rgba(0,0,0,0.7)',
                color: 'white',
                padding: '5px',
                borderRadius: '4px',
                pointerEvents: 'none',
                whiteSpace: 'pre-line',
                zIndex: 1000,
              }"
            >
              {{ tooltip }}
            </div>
          </v-col>
        </v-row>
      </v-container>
    </v-sheet>
  </v-card>
</template>

<script>
export default {
  data() {
    return {
      headers: [
        { text: "Timestamp", value: "ts_date_time" },
        { text: "Payload", value: "payload" },
      ],
      h_w: 800, // canvas width & height
      gridSize: 10, // 100 x 100 grid
      cellSize: 0, // computed
      ctx: null,
      dronePath: [],
      gridData: {}, // store cell info for hover
      tooltip: "",
      tooltipX: 0,
      tooltipY: 0,
    };
  },

  computed: {
    packets() {
      return this.$store.getters.allPackets;
    },
  },

  watch: {
    packets: {
      handler(newPackets) {
        this.updateGrid(newPackets);
      },
      deep: true,
    },
  },

  mounted() {
    this.cellSize = this.h_w / this.gridSize;
    this.ctx = this.$refs.gridCanvas.getContext("2d");
    this.drawGrid();
  },

  methods: {
    // Draw empty grid
    drawGrid() {
      const ctx = this.ctx;
      ctx.clearRect(0, 0, this.h_w, this.h_w);
      ctx.strokeStyle = "#eee";

      // vertical lines
      for (let i = 0; i <= this.gridSize; i++) {
        const x = i * this.cellSize;
        ctx.beginPath();
        ctx.moveTo(x, 0);
        ctx.lineTo(x, this.h_w);
        ctx.stroke();
      }

      // horizontal lines
      for (let i = 0; i <= this.gridSize; i++) {
        const y = i * this.cellSize;
        ctx.beginPath();
        ctx.moveTo(0, y);
        ctx.lineTo(this.h_w, y);
        ctx.stroke();
      }
    },

    // Parse single packet
    parsePacket(payload) {
      const cells = [];
      // split multiple cells
      const cellParts = payload.split(/\[ID\]:\d+/).filter(Boolean);

      cellParts.forEach((part) => {
        const cellMatch = part.match(/CELL \((\d+),\s*(\d+)\)/);
        const gpsMatch = part.match(/GPS (\d+),(\d+)/);
        if (!cellMatch) return;

        const x = parseInt(cellMatch[1]);
        const y = parseInt(cellMatch[2]);

        const gps = gpsMatch ? { lat: gpsMatch[1], lon: gpsMatch[2] } : null;

        // parse diseases: name: infected: healthy
        const diseaseMatches = [
          ...part.matchAll(/(\w+)\s*:\s*(\d+)\s*:\s*(\d+)/g),
        ];
        const diseases = diseaseMatches.map((m) => ({
          name: m[1],
          infected: parseInt(m[2]),
          healthy: parseInt(m[3]),
        }));

        cells.push({ x, y, gps, diseases });
      });

      return cells;
    },

    // Update grid with packets
    updateGrid(packets) {
      const ctx = this.ctx;
      this.dronePath = [];
      this.gridData = {};
      this.drawGrid();

      packets.forEach((packet) => {
        const cells = this.parsePacket(packet.payload);
        cells.forEach((cell) => {
          const { x, y, gps, diseases } = cell;
          if (x >= this.gridSize || y >= this.gridSize) return;

          // store for hover
          this.gridData[`${x},${y}`] = { gps, diseases };

          // compute color (simple: red intensity by infection ratio)
          let totalInfected = diseases.reduce((a, d) => a + d.infected, 0);
          let totalHealthy = diseases.reduce((a, d) => a + d.healthy, 0);

          let color = "#bdbdbd"; // default NO DATA
          if (totalInfected > 0) {
            const severity = totalInfected / (totalInfected + totalHealthy);
            color = `rgba(255,0,0,${Math.min(severity, 1)})`;
          } else if (diseases.length > 0) {
            color = "green";
          }

          const canvasY = this.h_w - (y + 1) * this.cellSize;

          // fill cell
          ctx.fillStyle = color;
          ctx.fillRect(
            x * this.cellSize,
            canvasY,
            this.cellSize,
            this.cellSize,
          );

          // draw label
          ctx.fillStyle = "black";
          ctx.font = `${this.cellSize / 3}px Arial`;
          ctx.textAlign = "center";
          ctx.textBaseline = "middle";
          ctx.fillText(
            `${x},${y}`,
            x * this.cellSize + this.cellSize / 2,
            canvasY + this.cellSize / 2,
          );

          this.dronePath.push({ x, y });
        });
      });

      // draw drone as triangle at last position
      if (this.dronePath.length) {
        const d = this.dronePath[this.dronePath.length - 1];
        const canvasY = this.h_w - (d.y + 1) * this.cellSize;

        const cx = d.x * this.cellSize + this.cellSize / 2;
        const cy = canvasY + this.cellSize / 2;
        const size = this.cellSize / 2; // size of triangle

        ctx.fillStyle = "yellow";
        ctx.beginPath();

        // Triangle pointing upward
        ctx.moveTo(cx, cy - size / 2); // top vertex
        ctx.lineTo(cx - size / 2, cy + size / 2); // bottom left
        ctx.lineTo(cx + size / 2, cy + size / 2); // bottom right
        ctx.closePath();
        ctx.fill();
      }
    },

    // popup
    onClickCell(e) {
      const canvas = this.$refs.gridCanvas;
      const rect = canvas.getBoundingClientRect();

      // mouse position relative to canvas
      const mx = e.clientX - rect.left;
      const my = e.clientY - rect.top;

      // compute column
      const col = Math.floor(mx / this.cellSize);

      // compute row (flip Y for canvas)
      let row = Math.floor((this.h_w - my) / this.cellSize);

      // clamp row/col to grid
      const clampedCol = Math.min(Math.max(col, 0), this.gridSize - 1);
      row = Math.min(Math.max(row, 0), this.gridSize - 1);

      const data = this.gridData[`${clampedCol},${row}`];

      if (data) {
        // tooltip position relative to container (canvas)
        this.tooltipX = mx + 5;
        this.tooltipY = my + 5;

        let txt = `GPS: ${
          data.gps ? data.gps.lat + "," + data.gps.lon : "N/A"
        }\n`;
        if (data.diseases.length) {
          txt += data.diseases
            .map(
              (d) => `${d.name}: infected ${d.infected}, healthy ${d.healthy}`,
            )
            .join("\n");
        } else {
          txt += "NO DATA";
        }
        this.tooltip = txt;
      } else {
        this.tooltip = "";
      }
    },
  },
};
</script>
