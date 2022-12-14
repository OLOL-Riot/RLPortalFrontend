<script setup lang="ts">
import type { IExerciseList } from "@/interfaces/IExercise";
import { getApi } from "@/api";
import { ref } from "vue";
import type { AxiosError, AxiosResponse } from "axios";
import type { ITest } from "@/interfaces/ITest";
import { useQuasar } from "quasar";
import type {
  IVerifyExercisePost,
  IVerifyTestPost,
} from "@/interfaces/IVerifyTestPost";
import type { IVerifyTestResponse } from "@/interfaces/IVerifyTestResponse";
import EditingToolBar from "./EditingToolBar.vue";
import { isAdministrator } from "@/roles";

const props = defineProps({
  testId: {
    type: String,
    required: true,
  },
  toEdit: {
    type: Function,
  },
});

const $q = useQuasar();

const exerciseList = ref({} as IExerciseList);
const test = ref({} as ITest);

interface IAnswersToExercisesArray {
  [index: string]: string;
}

interface IAnswersStatus {
  [index: string]: "red" | "green" | "blue";
}

const answersByExercises = ref({} as IAnswersToExercisesArray);
const answersStatus = ref({} as IAnswersStatus);

const result = ref("");
const solutionSent = ref(false);

getApi().then((api) =>
  api
    .get("api/Test/solve/" + props.testId)
    .then((response: AxiosResponse<ITest>) => {
      test.value = response.data;

      exerciseList.value = response.data.exercises.sort(
        (a, b) => a.serialNumber - b.serialNumber
      );

      response.data.exercises.forEach((exercise) => {
        answersStatus.value[exercise.id] = "blue";
      });
    })
    .catch((err: AxiosError) => {
      $q.notify({
        color: "red-5",
        textColor: "white",
        icon: "warning",
        message: err.message,
      });
    })
);

function checkAnsver() {
  let postData = {} as IVerifyTestPost;

  postData.testId = props.testId;
  postData.userAnswers = [];

  for (let exercise of exerciseList.value) {
    const chosenAnswer = answersByExercises.value[exercise.id];

    postData.userAnswers.push({
      exerciseId: exercise.id,
      chosenAnswer: typeof chosenAnswer != "undefined" ? chosenAnswer : null,
    });
  }

  getApi().then((api) =>
    api
      .post("api/VerifiedTest/", postData)
      .then((response: AxiosResponse<IVerifyTestResponse>) => {
        response.data.verifiedAnswers.forEach((question) => {
          if (question.isRight)
            answersStatus.value[question.exerciseId] = "green";
          else answersStatus.value[question.exerciseId] = "red";
        });

        if (response.data.points == response.data.maxPoints)
          $q.notify({
            color: "green-5",
            textColor: "white",
            icon: "done",
            message: "?????????????????? ??????????!",
          });
        /*
      $q.notify({
        color: "blue-5",
        textColor: "white",
        icon: "done",
        message: `???? ?????????????? ${response.data.points} ???? ${response.data.maxPoints} ????????????`,
      });*/

        result.value = `???? ?????????????? ${response.data.points} ???? ${response.data.maxPoints} ????????????`;
        solutionSent.value = true;
      })
  );
}
</script>

<template>
  <div class="container">
    <EditingToolBar v-if="isAdministrator()" :to-edit="toEdit" />
    <div class="q-pa-md column justify-center items-center">
      <h3 class="q-mb-xl">{{ test.name }}</h3>
      <q-card
        class="my-card q-mb-md"
        v-for="exercise in exerciseList"
        :key="exercise.id"
      >
        <q-card-section>
          <div class="text-h6">{{ exercise.description }}</div>
        </q-card-section>

        <q-separator />

        <div class="q-gutter-sm flex column q-py-sm">
          <q-radio
            class="q-pr-md"
            v-for="option in exercise.answers"
            :key="option"
            :color="answersStatus[exercise.id]"
            v-model="answersByExercises[exercise.id]"
            :val="option"
            :label="option"
            @click="answersStatus[exercise.id] = 'blue'"
          />
        </div>
      </q-card>
    </div>
    <div class="row q-pa-md">
      <q-btn
        class="q-mx-auto"
        color="green"
        label="?????????????????? ???????? ???? ????????????????"
        @click="checkAnsver()"
        :disable="solutionSent"
      />
    </div>
    <div class="result" :class="!solutionSent ? 'hidden' : ''">
      {{ result }}
    </div>
  </div>
</template>

<style scoped>
.my-card {
  width: 100%;
}

.result {
  width: max-content;
  margin-inline: auto;
}
</style>
