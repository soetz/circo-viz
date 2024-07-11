<script setup lang="ts">
import { computed, ref } from "vue";
import {
  circos as dataCircos,
  departements as dataDepartements,
  regions as dataRegions,
} from "../data";
import { csvParse } from "d3-dsv";
import { useMouse, useWindowSize } from "@vueuse/core";
import electedCsv from "../elected.csv?raw";

const SQUARE_SIZE = 8;
const GAP_SIZE = 0;
const NUMBER_OF_SQUARES_PER_SIDE = 32;

const { x: mouseX } = useMouse();
const { width: windowWidth } = useWindowSize();

const electedData = csvParse(electedCsv);

const sideOfMouse = computed(() => {
  return mouseX.value < windowWidth.value / 2 ? "left" : "right";
});

const sizeOfEdge = computed(() => {
  return (
    NUMBER_OF_SQUARES_PER_SIDE * SQUARE_SIZE +
    (NUMBER_OF_SQUARES_PER_SIDE + 1) * GAP_SIZE
  );
});

function coordinatesTransform(value: number) {
  return value * (SQUARE_SIZE + GAP_SIZE);
}

const displayCircos = computed(() => {
  return dataCircos.map((circo) => {
    const departement = dataDepartements.find(
      (dep) => circo.id.substring(0, dep.id.length) === dep.id,
    );

    const region = dataRegions.find(
      (region) => departement && region.departements.includes(departement?.id),
    );

    const electedInCirco = electedData.find((elected) => {
      return elected.idCirco === circo.id;
    });

    let elected;

    if (electedInCirco) {
      elected = {
        nuance: electedInCirco.nuance,
        firstName: electedInCirco.firstName,
        lastName: electedInCirco.lastName,
      };
    }

    return {
      ...circo,
      departement: departement?.id,
      elected,
      number: departement
        ? Number(circo.id.substring(departement?.id.length))
        : undefined,
      position: {
        x: circo.position.x + (departement?.position.x ?? 0),
        y: circo.position.y + (departement?.position.y ?? 0),
      },
      region: region?.id,
    };
  });
});

type Coordinate = {
  x: number;
  y: number;
};

function coordinatesContain(coordinates: Coordinate[], x: number, y: number) {
  return coordinates.some(
    (otherCoordinate) => otherCoordinate.x === x && otherCoordinate.y === y,
  );
}

function startBuildingPerimeter(coordinates: Coordinate[]) {
  const perimeter: Coordinate[] = [];
  let topLeftCornerCoordinate;
  let i = 0;

  while (!topLeftCornerCoordinate) {
    if (
      !coordinatesContain(
        coordinates,
        coordinates[i].x - 1,
        coordinates[i].y,
      ) &&
      !coordinatesContain(coordinates, coordinates[i].x, coordinates[i].y - 1)
    ) {
      topLeftCornerCoordinate = coordinates[i];
    }
    i++;
  }

  perimeter.push(topLeftCornerCoordinate);
  perimeter.push({
    x: topLeftCornerCoordinate.x + 1,
    y: topLeftCornerCoordinate.y,
  });

  return perimeter;
}

function chooseNextDirection(
  coordinates: Coordinate[],
  lastPerimeterCoordinate: Coordinate,
  lastDirection: "top" | "right" | "bottom" | "left",
) {
  if (lastDirection === "top") {
    if (
      coordinatesContain(
        coordinates,
        lastPerimeterCoordinate.x - 1,
        lastPerimeterCoordinate.y - 1,
      )
    ) {
      return "left";
    }
    if (
      coordinatesContain(
        coordinates,
        lastPerimeterCoordinate.x,
        lastPerimeterCoordinate.y - 1,
      )
    ) {
      return "top";
    }
    return "right";
  }

  if (lastDirection === "right") {
    if (
      coordinatesContain(
        coordinates,
        lastPerimeterCoordinate.x,
        lastPerimeterCoordinate.y - 1,
      )
    ) {
      return "top";
    }
    if (
      coordinatesContain(
        coordinates,
        lastPerimeterCoordinate.x,
        lastPerimeterCoordinate.y,
      )
    ) {
      return "right";
    }
    return "bottom";
  }

  if (lastDirection === "bottom") {
    if (
      coordinatesContain(
        coordinates,
        lastPerimeterCoordinate.x,
        lastPerimeterCoordinate.y,
      )
    ) {
      return "right";
    }
    if (
      coordinatesContain(
        coordinates,
        lastPerimeterCoordinate.x - 1,
        lastPerimeterCoordinate.y,
      )
    ) {
      return "bottom";
    }
    return "left";
  }

  if (lastDirection === "left") {
    if (
      coordinatesContain(
        coordinates,
        lastPerimeterCoordinate.x - 1,
        lastPerimeterCoordinate.y,
      )
    ) {
      return "bottom";
    }
    if (
      coordinatesContain(
        coordinates,
        lastPerimeterCoordinate.x - 1,
        lastPerimeterCoordinate.y - 1,
      )
    ) {
      return "left";
    }
  }
  return "top";
}

function buildPerimeterFromCoordinates(coordinates: Coordinate[]) {
  const builtPerimeter = startBuildingPerimeter(coordinates);

  let lastDirection: "top" | "right" | "bottom" | "left" = "right";
  let lastBuiltPerimeterCoordinate = builtPerimeter[builtPerimeter.length - 1];

  while (
    builtPerimeter[0].x !== lastBuiltPerimeterCoordinate.x ||
    builtPerimeter[0].y !== lastBuiltPerimeterCoordinate.y
  ) {
    const nextDirection = chooseNextDirection(
      coordinates,
      lastBuiltPerimeterCoordinate,
      lastDirection,
    );

    let nextBuiltCoordinate = {
      x: lastBuiltPerimeterCoordinate.x - 1,
      y: lastBuiltPerimeterCoordinate.y,
    };
    if (nextDirection === "top") {
      nextBuiltCoordinate = {
        x: lastBuiltPerimeterCoordinate.x,
        y: lastBuiltPerimeterCoordinate.y - 1,
      };
    }
    if (nextDirection === "right") {
      nextBuiltCoordinate = {
        x: lastBuiltPerimeterCoordinate.x + 1,
        y: lastBuiltPerimeterCoordinate.y,
      };
    }
    if (nextDirection === "bottom") {
      nextBuiltCoordinate = {
        x: lastBuiltPerimeterCoordinate.x,
        y: lastBuiltPerimeterCoordinate.y + 1,
      };
    }

    builtPerimeter.push(nextBuiltCoordinate);
    lastBuiltPerimeterCoordinate = nextBuiltCoordinate;
    lastDirection = nextDirection;
  }

  return builtPerimeter;
}

function buildPolygonPointsFromCoordinates(coordinates: Coordinate[]) {
  return buildPerimeterFromCoordinates(coordinates)
    .map((coordinate) => {
      return `${coordinatesTransform(coordinate.x) - 0.5 * GAP_SIZE},${coordinatesTransform(coordinate.y) - 0.5 * GAP_SIZE}`;
    })
    .join(" ");
}

const departementsWithCoordinates = computed(() => {
  return dataDepartements.map((dep) => {
    const circosCoordinates = dataCircos
      .filter((circo) => {
        return circo.id.substring(0, dep.id.length) === dep.id;
      })
      .map((circo) => {
        return {
          x: circo.position.x + (dep?.position.x ?? 0),
          y: circo.position.y + (dep?.position.y ?? 0),
        };
      });

    return {
      id: dep.id,
      determinant: dep.determinant,
      name: dep.name,
      coordinates: circosCoordinates,
    };
  });
});

function groupCircos(coordinates: Coordinate[]) {
  coordinates = [...coordinates];

  let groups: Coordinate[][] = [];

  while (coordinates.length > 0) {
    const currentCoordinate = coordinates.pop()!;

    const groupsWhereCoordinateBelongs = groups.filter((group) =>
      group.some(
        (coordinateFromGroup) =>
          (coordinateFromGroup.x === currentCoordinate.x + 1 &&
            coordinateFromGroup.y === currentCoordinate.y) ||
          (coordinateFromGroup.x === currentCoordinate.x - 1 &&
            coordinateFromGroup.y === currentCoordinate.y) ||
          (coordinateFromGroup.x === currentCoordinate.x &&
            coordinateFromGroup.y === currentCoordinate.y + 1) ||
          (coordinateFromGroup.x === currentCoordinate.x &&
            coordinateFromGroup.y === currentCoordinate.y - 1),
      ),
    );

    if (groupsWhereCoordinateBelongs.length < 1) {
      groups.push([currentCoordinate]);
    } else {
      const otherGroups = groups.filter(
        (group) => !groupsWhereCoordinateBelongs.includes(group),
      );

      const mergedGroup = [currentCoordinate].concat(
        ...groupsWhereCoordinateBelongs,
      );

      otherGroups.push(mergedGroup);

      groups = otherGroups;
    }
  }

  return groups;
}

const displayDepartements = computed(() => {
  return departementsWithCoordinates.value.map((dep) => {
    const groups = groupCircos(dep.coordinates);
    const perimeters: string[] = [];

    for (const group of groups) {
      perimeters.push(buildPolygonPointsFromCoordinates(group));
    }

    return {
      ...dep,
      perimeters,
    };
  });
});

const displayRegions = computed(() => {
  return dataRegions.map((reg) => {
    const circosCoordinates = departementsWithCoordinates.value
      .filter((dep) => {
        return reg.departements.includes(dep.id);
      })
      .map((dep) => {
        return dep.coordinates;
      })
      .flat(1);

    const groups = groupCircos(circosCoordinates);
    const perimeters: string[] = [];

    for (const group of groups) {
      perimeters.push(buildPolygonPointsFromCoordinates(group));
    }

    return {
      id: reg.id,
      perimeters,
    };
  });
});

const hoveredDepartement = ref<string | null>(null);
const clickedDepartement = ref<string | null>(null);

function setHoveredDepartement(departementId?: string | null) {
  hoveredDepartement.value = departementId ?? null;
}

function toggleClickedDepartement(departementId?: string | null) {
  if (departementId === clickedDepartement.value) {
    clickedDepartement.value = null;
  } else {
    clickedDepartement.value = departementId ?? null;
  }
}

const focusedDepartement = computed(
  () => clickedDepartement.value ?? hoveredDepartement.value,
);

const focusedDepartementDetails = computed(() => {
  const departement = dataDepartements.find(
    (dep) => dep.id === focusedDepartement.value,
  );

  if (departement) {
    const circos = [
      ...dataCircos
        .filter(
          (circo) =>
            circo.id.substring(0, departement?.id.length) === departement?.id,
        )
        .map((circo) => ({
          ...circo,
          elected: electedData.find((elected) => elected.idCirco === circo.id),
          number: Number(circo.id.substring(departement?.id.length)),
        })),
    ].sort((circoA, circoB) => {
      return circoA.number - circoB.number;
    });

    return {
      ...departement,
      circos,
    };
  }

  return null;
});
</script>

<template>
  <div class="container">
    <div
      :class="['focused-departement', { 'right-side': sideOfMouse === 'left' }]"
      v-if="focusedDepartement"
    >
      <h1
        >Circonscriptions{{ focusedDepartementDetails?.determinant
        }}{{ focusedDepartementDetails?.name }}&nbsp;({{
          focusedDepartementDetails?.id
        }})</h1
      >
      <ul>
        <li
          v-for="circo of focusedDepartementDetails?.circos"
          :key="circo.id"
          ><div
            :class="['nuance-preview', circo.elected?.nuance.toLowerCase()]"
          ></div
          ><span
            >#{{ circo.number }}&thinsp;: {{ circo.elected?.firstName }}
            {{ circo.elected?.lastName }}, {{ circo.elected?.nuance }}</span
          ></li
        >
      </ul>
    </div>
    <svg
      xmlns="http://www.w3.org/2000/svg"
      :viewBox="`0 0 ${sizeOfEdge} ${sizeOfEdge}`"
    >
      <rect
        class="background"
        x="0"
        y="0"
        :width="sizeOfEdge"
        :height="sizeOfEdge"
      />
      <g>
        <template
          v-for="circo of displayCircos"
          :key="circo.id"
        >
          <rect
            :class="[
              `circo dep-${circo.departement} reg-${circo.region}`,
              circo.elected
                ? `elec-${circo.elected?.nuance.toLowerCase()}`
                : '',
            ]"
            :x="coordinatesTransform(circo.position.x)"
            :y="coordinatesTransform(circo.position.y)"
            :width="SQUARE_SIZE + 0.2"
            :height="SQUARE_SIZE + 0.2"
            @mouseenter="setHoveredDepartement(circo.departement)"
            @mouseleave="setHoveredDepartement(null)"
            @click="toggleClickedDepartement(circo.departement)"
          />
          <text
            v-if="circo.departement === focusedDepartement"
            class="circo-number"
            :x="coordinatesTransform(circo.position.x) + 0.5 * SQUARE_SIZE"
            :y="
              coordinatesTransform(circo.position.y) + 0.5 * SQUARE_SIZE + 1.5
            "
          >
            {{ circo.number }}
          </text>
        </template>
      </g>
      <g>
        <template v-for="departement of displayDepartements">
          <polygon
            v-for="(perimeter, index) of departement.perimeters"
            :key="`${departement.id}-${index}`"
            class="dep-stroke"
            :points="perimeter"
          />
        </template>
      </g>
      <g>
        <template v-for="region of displayRegions">
          <polygon
            v-for="(perimeter, index) of region.perimeters"
            :key="`${region.id}-${index}`"
            class="reg-stroke"
            :points="perimeter"
          />
        </template>
      </g>
      <template v-for="departement of displayDepartements">
        <template v-if="departement.id === focusedDepartement">
          <polygon
            v-for="(perimeter, index) of departement.perimeters"
            :key="`${departement.id}-${index}-focused`"
            class="dep-stroke focused"
            :points="perimeter"
          />
        </template>
      </template>
    </svg>
  </div>
</template>

<style scoped>
.container {
  display: flex;
  align-items: end;
  justify-content: end;

  width: 100%;
  height: 100%;

  text-align: right;

  --nuance-exd: oklch(30% 0.08 48);
  --nuance-rn: oklch(25% 0.16 272);
  --nuance-uxd: oklch(40% 0.16 272);
  --nuance-lr: oklch(50% 0.16 272);
  --nuance-udi: oklch(85% 0.14 190);
  --nuance-dvd: oklch(80% 0.05 272);
  --nuance-hor: oklch(65% 0.14 190);
  --nuance-dvc: oklch(90% 0.05 75);
  --nuance-div: oklch(65% 0.01 100);
  --nuance-ens: oklch(80% 0.16 75);
  --nuance-reg: oklch(65% 0.2 40);
  --nuance-soc: oklch(50% 0.2 25);
  --nuance-dvg: oklch(75% 0.2 330);
  --nuance-eco: oklch(75% 0.3 145);
  --nuance-ug: oklch(45% 0.2 330);
}

.focused-departement {
  position: fixed;
  top: 0;
  left: 0;

  padding-top: 16px;
  padding-right: 16px;
  padding-bottom: 16px;
  padding-left: 16px;

  background-color: rgb(0 0 0 / 0.8);
  backdrop-filter: blur(4px);

  text-align: left;

  max-width: max(50%, 600px);

  pointer-events: none;
}
.focused-departement.right-side {
  left: auto;
  right: 0;
}

.focused-departement h1 {
  margin-top: 0;
  margin-right: 0;
  margin-bottom: 0;
  margin-left: 0;

  font-size: 1.2rem;
}

.focused-departement ul {
  display: flex;
  flex-direction: column;

  margin-top: 12px;
  margin-right: 0;
  margin-bottom: 0;
  margin-left: 0;
  padding-top: 0;
  padding-right: 0;
  padding-bottom: 0;
  padding-left: 0;

  list-style-type: none;

  text-align: left;
}

.focused-departement li {
  display: flex;
  align-items: center;
  gap: 8px;
}

.nuance-preview {
  width: 1rem;
  height: 1rem;

  border-top-color: black;
  border-top-width: 1px;
  border-top-style: solid;
  border-right-color: black;
  border-right-width: 1px;
  border-right-style: solid;
  border-bottom-color: black;
  border-bottom-width: 1px;
  border-bottom-style: solid;
  border-left-color: black;
  border-left-width: 1px;
  border-left-style: solid;
}

.nuance-preview.exd {
  background-color: var(--nuance-exd);
}
.nuance-preview.rn {
  background-color: var(--nuance-rn);
}
.nuance-preview.uxd {
  background-color: var(--nuance-uxd);
}
.nuance-preview.lr {
  background-color: var(--nuance-lr);
}
.nuance-preview.udi {
  background-color: var(--nuance-udi);
}
.nuance-preview.dvd {
  background-color: var(--nuance-dvd);
}
.nuance-preview.hor {
  background-color: var(--nuance-hor);
}
.nuance-preview.dvc {
  background-color: var(--nuance-dvc);
}
.nuance-preview.div {
  background-color: var(--nuance-div);
}
.nuance-preview.ens {
  background-color: var(--nuance-ens);
}
.nuance-preview.reg {
  background-color: var(--nuance-reg);
}
.nuance-preview.soc {
  background-color: var(--nuance-soc);
}
.nuance-preview.dvg {
  background-color: var(--nuance-dvg);
}
.nuance-preview.eco {
  background-color: var(--nuance-eco);
}
.nuance-preview.ug {
  background-color: var(--nuance-ug);
}

svg {
  width: 100vmin;
  height: 100vmin;
}

.background {
  fill: transparent;
}

.circo {
  fill: oklch(75% 0 0);
}

.circo-number {
  font-size: 4px;
  text-anchor: middle;

  fill: white;

  pointer-events: none;
}

.dep-stroke,
.reg-stroke {
  fill: none;
  stroke: black;
  stroke-linejoin: round;

  pointer-events: none;
}

.dep-stroke {
  stroke-width: 0.5;
}
.dep-stroke.focused {
  stroke: oklch(65% 0.28 25);
  stroke-width: 2;
}

.reg-stroke {
  stroke-width: 1.5;
}

.elec-exd {
  fill: var(--nuance-exd);
}
.elec-rn {
  fill: var(--nuance-rn);
}
.elec-uxd {
  fill: var(--nuance-uxd);
}
.elec-lr {
  fill: var(--nuance-lr);
}
.elec-udi {
  fill: var(--nuance-udi);
}
.elec-dvd {
  fill: var(--nuance-dvd);
}
.elec-hor {
  fill: var(--nuance-hor);
}
.elec-dvc {
  fill: var(--nuance-dvc);
}
.elec-div {
  fill: var(--nuance-div);
}
.elec-ens {
  fill: var(--nuance-ens);
}
.elec-reg {
  fill: var(--nuance-reg);
}
.elec-soc {
  fill: var(--nuance-soc);
}
.elec-dvg {
  fill: var(--nuance-dvg);
}
.elec-eco {
  fill: var(--nuance-eco);
}
.elec-ug {
  fill: var(--nuance-ug);
}
</style>
