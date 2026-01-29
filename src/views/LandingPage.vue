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

      /* ================= CANVAS / GRID ================= */
      h_w: 800, // canvas width & height
      gridRows: 20, // number of rows (Y axis)
      gridCols: 20, // number of columns (X axis)
      cellW: 0, // computed cell width
      cellH: 0, // computed cell height

      /* ================= STATE ================= */
      ctx: null,
      dronePath: [],
      gridData: {},

      /* ================= TOOLTIP ================= */
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
    this.cellW = this.h_w / this.gridCols;
    this.cellH = this.h_w / this.gridRows;
    this.ctx = this.$refs.gridCanvas.getContext("2d");
    this.drawGrid();
  },

  methods: {
    /* ================= DRAW GRID ================= */
    drawGrid() {
      const ctx = this.ctx;
      ctx.clearRect(0, 0, this.h_w, this.h_w);
      ctx.strokeStyle = "#eee";

      // vertical lines (columns)
      for (let c = 0; c <= this.gridCols; c++) {
        const x = c * this.cellW;
        ctx.beginPath();
        ctx.moveTo(x, 0);
        ctx.lineTo(x, this.h_w);
        ctx.stroke();
      }

      // horizontal lines (rows)
      for (let r = 0; r <= this.gridRows; r++) {
        const y = r * this.cellH;
        ctx.beginPath();
        ctx.moveTo(0, y);
        ctx.lineTo(this.h_w, y);
        ctx.stroke();
      }
    },

    /* ================= PARSE PAYLOAD ================= */
    parsePacket(payload) {
      const cells = [];
      const parts = payload.split(/\[ID\]:\d+/).filter(Boolean);

      parts.forEach((part) => {
        const cellMatch = part.match(/CELL \((\d+),\s*(\d+)\)/);
        if (!cellMatch) return;

        const x = parseInt(cellMatch[1]);
        const y = parseInt(cellMatch[2]);

        const gpsMatch = part.match(/GPS (\d+),(\d+)/);
        const gps = gpsMatch ? { lat: gpsMatch[1], lon: gpsMatch[2] } : null;

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

    /* ================= UPDATE GRID ================= */
    updateGrid(packets) {
      const ctx = this.ctx;
      this.gridData = {};
      this.dronePath = [];
      this.drawGrid();

      packets.forEach((packet) => {
        const cells = this.parsePacket(packet.payload);

        cells.forEach(({ x, y, gps, diseases }) => {
          if (x >= this.gridCols || y >= this.gridRows) return;

          this.gridData[`${x},${y}`] = { gps, diseases };

          const infected = diseases.reduce((a, d) => a + d.infected, 0);
          const healthy = diseases.reduce((a, d) => a + d.healthy, 0);

          let color = "#bdbdbd";
          if (infected > 0) {
            const sev = infected / (infected + healthy);
            color = `rgba(255,0,0,${Math.min(sev, 1)})`;
          } else if (!infected) {
            color = "green";
          }

          const canvasY = this.h_w - (y + 1) * this.cellH;

          ctx.fillStyle = color;
          ctx.fillRect(x * this.cellW, canvasY, this.cellW, this.cellH);

          ctx.fillStyle = "black";
          ctx.font = `${Math.min(this.cellW, this.cellH) / 3}px Arial`;
          ctx.textAlign = "center";
          ctx.textBaseline = "middle";
          ctx.fillText(
            `${x},${y}`,
            x * this.cellW + this.cellW / 2,
            canvasY + this.cellH / 2,
          );

          this.dronePath.push({ x, y });
        });
      });

      this.drawDrone(ctx);
    },

    /* ================= DRAW DRONE (TRIANGLE) ================= */
    drawDrone(ctx) {
      if (!this.dronePath.length) return;

      const d = this.dronePath[this.dronePath.length - 1];
      const canvasY = this.h_w - (d.y + 1) * this.cellH;

      const cx = d.x * this.cellW + this.cellW / 2;
      const cy = canvasY + this.cellH / 2;
      const size = Math.min(this.cellW, this.cellH) * 0.6;

      ctx.fillStyle = "yellow";
      ctx.beginPath();
      ctx.moveTo(cx, cy - size / 2);
      ctx.lineTo(cx - size / 2, cy + size / 2);
      ctx.lineTo(cx + size / 2, cy + size / 2);
      ctx.closePath();
      ctx.fill();
    },

    /* ================= TOOLTIP CLICK ================= */
    onClickCell(e) {
      const rect = this.$refs.gridCanvas.getBoundingClientRect();
      const mx = e.clientX - rect.left;
      const my = e.clientY - rect.top;

      const col = Math.floor(mx / this.cellW);
      const row = Math.floor((this.h_w - my) / this.cellH);

      if (col < 0 || col >= this.gridCols || row < 0 || row >= this.gridRows) {
        this.tooltip = "";
        return;
      }

      const data = this.gridData[`${col},${row}`];
      if (!data) {
        this.tooltip = "";
        return;
      }

      this.tooltipX = mx + 6;
      this.tooltipY = my + 6;

      let txt = `CELL: (${col} , ${row})\n`;
      txt += `GPS: ${data.gps ? `${data.gps.lat} , ${data.gps.lon}` : "N/A"}\n`;

      if (data.diseases.length) {
        txt += data.diseases
          .map(
            (d) =>
              `DISEASE: ${d.name} | Infected ${d.infected} | Healthy ${d.healthy}`,
          )
          .join("\n");
      } else {
        txt += "NO DISEASES DETECTED";
      }

      this.tooltip = txt;
    },
  },
};
</script>
