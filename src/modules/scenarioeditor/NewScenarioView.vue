<script setup lang="ts">
import { reactive, ref } from "vue";
import { ExternalLinkIcon } from "@lucide/vue";
import FormCard from "@/components/FormCard.vue";
import InputGroup from "@/components/InputGroup.vue";
import SimpleMarkdownInput from "@/components/SimpleMarkdownInput.vue";
import TimezoneSelect from "@/components/TimezoneSelect.vue";
import { useYMDElements } from "@/composables/scenarioTime";
import RadioGroupList from "@/components/RadioGroupList.vue";
import BaseButton from "@/components/BaseButton.vue";
import { useRouter } from "vue-router";
import { MAP_EDIT_MODE_ROUTE } from "@/router/names";
import { useScenario } from "@/scenariostore";
import { createEmptyScenario } from "@/scenariostore/io";
import ToggleField from "@/components/ToggleField.vue";
import type { ScenarioInfo, SideData, UnitSymbolOptions } from "@/types/scenarioModels";
import { SID, type SidValue } from "@/symbology/values";
import { nanoid } from "@/utils";
import { Sidc } from "@/symbology/sidc";
import StandardIdentitySelect from "@/components/StandardIdentitySelect.vue";
import SimpleDivider from "@/components/SimpleDivider.vue";
import type { SymbolItem, SymbolValue } from "@/types/constants";
import { echelonItems } from "@/symbology/helpers";
import { useIndexedDb } from "@/scenariostore/localdb";
import SymbolCodeSelect from "@/components/SymbolCodeSelect.vue";
import { Button } from "@/components/ui/button";
import NewMilitarySymbol from "@/components/NewMilitarySymbol.vue";
import { stableSymbologyStandardOptions } from "@/symbology/standards";

const router = useRouter();
const { scenario } = useScenario();

interface RootUnit {
  rootUnitName?: string;
  rootUnitSidc?: string;
  rootUnitEchelon?: string;
  rootUnitIcon?: string;
}

interface InitialSideData extends SideData {
  symbolOptions: UnitSymbolOptions;
  units: RootUnit[];
}

interface NewScenarioForm extends ScenarioInfo {
  sides: InitialSideData[];
}

const noInitialOrbat = ref(false);

const newScenario = ref(
  createEmptyScenario({ addGroups: true, symbologyStandard: "app6d" }),
);
const timeZone = ref(newScenario.value.timeZone || "UTC");
const { year, month, day, hour, minute, resDateTime } = useYMDElements({
  timestamp: newScenario.value.startTime!,
  isLocal: true,
  timeZone,
});

const form = reactive<NewScenarioForm>({
  name: "新场景",
  description: "",
  sides: [
    {
      name: "阵营 1",
      standardIdentity: SID.Friend,
      symbolOptions: {},
      units: [{ rootUnitName: "指挥部", rootUnitEchelon: "18", rootUnitIcon: "121100" }],
    },
    {
      name: "阵营 2",
      standardIdentity: SID.Hostile,
      symbolOptions: {},
      units: [{ rootUnitName: "指挥部", rootUnitEchelon: "18", rootUnitIcon: "121100" }],
    },
  ],
});

async function create() {
  const startTime = resDateTime.value.valueOf();
  newScenario.value.startTime = startTime;
  newScenario.value.name = form.name;
  newScenario.value.description = form.description;
  newScenario.value.layerStack = [
    { name: "要素", id: nanoid(), kind: "overlay", items: [] },
  ];
  newScenario.value.timeZone = timeZone.value;

  scenario.value.io.loadFromObject(newScenario.value);
  scenario.value.time.setCurrentTime(startTime);
  const { state, clearUndoRedoStack } = scenario.value.store;
  const {
    unitActions,
    helpers: { getSideById },
  } = scenario.value;
  if (!noInitialOrbat.value) {
    form.sides.forEach((sideData) => {
      const { units, ...rest } = sideData;
      const sideId = unitActions.addSide(rest, { markAsNew: false });
      const parentId = getSideById(sideId).groups[0];
      sideData.units.forEach((u) => {
        const sidc = new Sidc("10031000000000000000");
        sidc.standardIdentity = sideData.standardIdentity;
        sidc.emt = u.rootUnitEchelon || "00";
        sidc.mainIcon = u.rootUnitIcon || "000000";
        unitActions.addUnit(
          {
            id: nanoid(),
            name: u.rootUnitName ?? "单位",
            sidc: sidc.toString(),
            subUnits: [],
            _pid: "nn",
            _sid: "nn",
            _gid: "nn",
            equipment: [],
            personnel: [],
          },
          parentId,
        );
      });
    });
  }
  clearUndoRedoStack();

  const { addScenario } = await useIndexedDb();
  const scenarioId = await addScenario(scenario.value.io.serializeToObject());

  await router.push({ name: MAP_EDIT_MODE_ROUTE, params: { scenarioId } });
}

const icons: SymbolValue[] = [
  { code: "000000", text: "未指定" },
  { code: "110000", text: "指挥与控制" },
  { code: "121100", text: "步兵" },
  { code: "121000", text: "合成兵种" },
  { code: "121102", text: "机械化" },
  { code: "130300", text: "炮兵" },
  { code: "120500", text: "装甲" },
  { code: "160600", text: "战斗勤务支援" },
];

function iconItems(sid: SidValue) {
  return icons.map(({ code, text }): SymbolItem => {
    return {
      code,
      text,
      sidc: "100" + sid + "10" + "00" + "00" + code + "0000",
    };
  });
}

function unitSidc(
  { rootUnitEchelon, rootUnitIcon }: RootUnit,
  { standardIdentity }: SideData,
) {
  return "100" + standardIdentity + "10" + "00" + rootUnitEchelon + rootUnitIcon + "0000";
}

function addSide() {
  form.sides.push({
    name: "阵营",
    standardIdentity: SID.Friend,
    symbolOptions: {},
    units: [{ rootUnitName: "指挥部", rootUnitEchelon: "18", rootUnitIcon: "121000" }],
  });
}

function addRootUnit(side: InitialSideData) {
  side.units.push({ rootUnitName: "指挥部", rootUnitEchelon: "18", rootUnitIcon: "121000" });
}

function removeUnit(side: InitialSideData, unit: RootUnit) {
  const idx = side.units.indexOf(unit);
  if (idx >= 0) side.units.splice(idx, 1);
}
</script>

<template>
  <div class="min-h-screen py-10">
    <header>
      <div class="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
        <h1 class="text-heading text-3xl leading-tight font-bold">创建新场景</h1>
        <div class="prose dark:prose-invert mt-4">
          <p>
            如果需要，你可以在这里为场景提供一些初始数据。这些设置之后随时都可以更改。
          </p>
        </div>
      </div>
    </header>
    <div class="mx-auto my-10 max-w-7xl sm:px-6 lg:px-8">
      <form class="mt-6 space-y-6" @submit.prevent="create()">
        <div class="flex items-center justify-between space-x-3 px-4 sm:px-0">
          <Button as-child variant="link"
            ><a href="https://docs.orbat-mapper.app/guide/getting-started" target="_blank"
              >查看文档 <ExternalLinkIcon />
            </a>
          </Button>
          <BaseButton primary type="submit">创建场景</BaseButton>
        </div>
        <FormCard
          class=""
          label="场景基本信息"
          description="为你的场景提供一个名称和描述。"
        >
          <InputGroup label="名称" v-model="form.name" id="name-input" autofocus />

          <SimpleMarkdownInput
            label="描述"
            v-model="form.description"
            description="使用 Markdown 语法进行格式化"
          />
        </FormCard>
        <FormCard label="初始作战序列 (ORBAT)">
          <template #description> 阵营和根单位。</template>
          <div>
            <ToggleField v-model="noInitialOrbat"
              >稍后添加阵营和根单位
            </ToggleField>
          </div>
          <template v-if="!noInitialOrbat">
            <div
              v-for="(sideData, idx) in form.sides"
              :key="idx"
              class="relative rounded-md border p-4"
            >
              <div class="grid gap-4 md:grid-cols-2">
                <InputGroup v-model="sideData.name" label="阵营名称" />
              </div>
              <StandardIdentitySelect
                v-model="sideData.standardIdentity"
                v-model:fill-color="sideData.symbolOptions.fillColor"
              />
              <SimpleDivider class="mt-4 mb-4">根单位</SimpleDivider>
              <div class="space-y-6">
                <template v-for="(unit, i) in sideData.units" :key="i">
                  <div class="flex items-end gap-4 md:grid md:grid-cols-2">
                    <InputGroup label="根单位名称" v-model="unit.rootUnitName" />
                    <NewMilitarySymbol
                      :size="32"
                      :sidc="unitSidc(unit, sideData)"
                      :options="{ ...sideData.symbolOptions, outlineWidth: 8 }"
                    />
                  </div>
                  <div class="mt-4 grid gap-4 md:grid-cols-2">
                    <SymbolCodeSelect
                      class=""
                      label="主图标"
                      v-model="unit.rootUnitIcon"
                      :items="iconItems(sideData.standardIdentity)"
                      :symbol-options="sideData.symbolOptions"
                    />
                    <SymbolCodeSelect
                      class="w-full"
                      label="编制层级"
                      v-model="unit.rootUnitEchelon"
                      :items="echelonItems(sideData.standardIdentity)"
                      :symbol-options="sideData.symbolOptions"
                    />
                  </div>
                  <p class="text-muted-foreground text-sm">
                    如果你找不到合适的图标，不用担心，之后可以随时更改。
                  </p>
                  <SimpleDivider v-if="i < sideData.units.length - 1" />
                </template>
              </div>
              <footer class="mt-6 flex justify-end gap-x-2">
                <Button
                  variant="link"
                  type="button"
                  size="sm"
                  :disabled="!sideData.units.length"
                  @click="removeUnit(sideData, sideData.units[sideData.units.length - 1])"
                >
                  移除单位
                </Button>
                <span class="text-border">|</span>
                <Button
                  type="button"
                  variant="link"
                  size="sm"
                  @click="addRootUnit(sideData)"
                >
                  + 添加根单位
                </Button>
              </footer>
              <Button
                variant="link"
                size="sm"
                type="button"
                v-if="idx === form.sides.length - 1"
                @click="form.sides.pop()"
              >
                移除阵营
              </Button>
            </div>
            <footer class="mt-6 flex justify-end">
              <Button type="button" variant="link" size="sm" @click="addSide()">
                + 添加阵营
              </Button>
            </footer>
          </template>
        </FormCard>
        <FormCard label="场景开始时间">
          <template #description>
            <p>选择开始时间和时区。</p>
          </template>
          <TimezoneSelect label="时区" v-model="timeZone" />

          <div class="grid grid-cols-3 gap-6">
            <InputGroup label="年" type="number" v-model="year" />
            <InputGroup label="月" type="number" v-model="month" />
            <InputGroup label="日" type="number" v-model="day" />
          </div>
          <div class="grid grid-cols-2 gap-6">
            <InputGroup label="时" v-model="hour" type="number" min="0" max="23" />
            <InputGroup label="分" v-model="minute" type="number" min="0" max="59" />
          </div>
          <p class="text-muted-foreground font-mono">{{ resDateTime.format() }}</p>
        </FormCard>
        <FormCard
          label="符号体系标准"
          description="选择你偏好的符号体系标准。"
        >
          <RadioGroupList
            :items="stableSymbologyStandardOptions"
            v-model="newScenario.symbologyStandard"
          />
        </FormCard>
        <div class="flex justify-end space-x-3 px-4 sm:px-0">
          <Button type="submit">创建场景</Button>
          <Button asChild variant="secondary"
            ><RouterLink to="/">取消</RouterLink></Button
          >
        </div>
      </form>
    </div>
  </div>
</template>
